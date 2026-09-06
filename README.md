"""
Complete E-Commerce Backend - Ghanaian Food Crops
Specialized for Cassava, Maize, Potatoes, Yam, Rice, Wheat
"""

import os
import re
import json
import secrets
import jwt
import bcrypt
import psycopg2
from psycopg2.extras import RealDictCursor
from datetime import datetime, timedelta
from functools import wraps
from flask import Flask, request, jsonify, send_from_directory
from flask_cors import CORS
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
from dotenv import load_dotenv
import requests
from twilio.rest import Client
import redis
from contextlib import contextmanager

load_dotenv()

app = Flask(__name__, static_folder='.', static_url_path='')
CORS(app)

limiter = Limiter(get_remote_address, app=app, default_limits=["200 per day", "50 per hour"], storage_uri="memory://")

SECRET_KEY = os.getenv('SECRET_KEY', 'your-secret-key-change-in-production')
DATABASE_URL = os.getenv('DATABASE_URL', 'postgresql://postgres:password@localhost:5432/ecom_ghana')
REDIS_URL = os.getenv('REDIS_URL', 'redis://localhost:6379/0')
TWILIO_ACCOUNT_SID = os.getenv('TWILIO_ACCOUNT_SID')
TWILIO_AUTH_TOKEN = os.getenv('TWILIO_AUTH_TOKEN')
TWILIO_WHATSAPP_NUMBER = os.getenv('TWILIO_WHATSAPP_NUMBER', '+14155238886')
JWT_EXPIRATION = int(os.getenv('JWT_EXPIRATION', 86400))

try:
    redis_client = redis.from_url(REDIS_URL)
except:
    redis_client = None

# ==================== DATABASE ====================
class Database:
    @staticmethod
    @contextmanager
    def get_connection():
        conn = psycopg2.connect(DATABASE_URL)
        try:
            yield conn
        finally:
            conn.close()
    
    @staticmethod
    @contextmanager
    def get_cursor(commit=False):
        with Database.get_connection() as conn:
            cursor = conn.cursor(cursor_factory=RealDictCursor)
            try:
                yield cursor
                if commit:
                    conn.commit()
            except Exception as e:
                conn.rollback()
                raise e
            finally:
                cursor.close()

    @staticmethod
    def execute_query(query, params=None, fetch_one=False, fetch_all=False, commit=False):
        with Database.get_cursor(commit) as cursor:
            cursor.execute(query, params or ())
            if fetch_one:
                return cursor.fetchone()
            if fetch_all:
                return cursor.fetchall()
            return None

    @staticmethod
    def init_db():
        """Initialize database with food crop focus"""
        
        tables = [
            """
            CREATE TABLE IF NOT EXISTS users (
                id SERIAL PRIMARY KEY,
                email VARCHAR(255) UNIQUE NOT NULL,
                password_hash TEXT NOT NULL,
                full_name VARCHAR(100) NOT NULL,
                phone VARCHAR(20) NOT NULL,
                address TEXT,
                is_admin BOOLEAN DEFAULT FALSE,
                email_verified BOOLEAN DEFAULT FALSE,
                verification_token TEXT,
                reset_token TEXT,
                reset_token_expiry TIMESTAMP,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
            """,
            """
            CREATE TABLE IF NOT EXISTS crop_categories (
                id SERIAL PRIMARY KEY,
                name VARCHAR(50) UNIQUE NOT NULL,
                description TEXT,
                icon VARCHAR(50),
                growing_season VARCHAR(100),
                harvest_season VARCHAR(100),
                storage_life_days INTEGER
            )
            """,
            """
            CREATE TABLE IF NOT EXISTS products (
                id SERIAL PRIMARY KEY,
                name VARCHAR(200) NOT NULL,
                description TEXT,
                price_per_kg DECIMAL(10,2) NOT NULL CHECK (price_per_kg >= 0),
                category_id INTEGER REFERENCES crop_categories(id),
                stock_kg INTEGER DEFAULT 0 CHECK (stock_kg >= 0),
                image_url TEXT,
                sku VARCHAR(50) UNIQUE,
                is_active BOOLEAN DEFAULT TRUE,
                
                -- Crop specific fields
                crop_variety VARCHAR(100),
                growing_region VARCHAR(100),
                harvest_date DATE,
                planting_season VARCHAR(50),
                harvest_season VARCHAR(50),
                days_to_maturity INTEGER,
                storage_instructions TEXT,
                nutritional_info TEXT,
                
                -- Quality grading
                grade VARCHAR(20) CHECK (grade IN ('Premium', 'Grade 1', 'Grade 2', 'Standard')),
                moisture_content DECIMAL(5,2),
                purity_percentage DECIMAL(5,2),
                
                -- Bulk pricing
                is_bulk BOOLEAN DEFAULT FALSE,
                bulk_min_kg INTEGER,
                bulk_price_per_kg DECIMAL(10,2),
                
                -- Sustainability
                is_organic BOOLEAN DEFAULT FALSE,
                is_fair_trade BOOLEAN DEFAULT FALSE,
                carbon_footprint_kg DECIMAL(8,2),
                
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
            """,
            """
            CREATE TABLE IF NOT EXISTS crop_seasons (
                id SERIAL PRIMARY KEY,
                crop_category_id INTEGER REFERENCES crop_categories(id),
                region VARCHAR(100),
                planting_start_month INTEGER CHECK (planting_start_month >= 1 AND planting_start_month <= 12),
                planting_end_month INTEGER CHECK (planting_end_month >= 1 AND planting_end_month <= 12),
                harvest_start_month INTEGER CHECK (harvest_start_month >= 1 AND harvest_start_month <= 12),
                harvest_end_month INTEGER CHECK (harvest_end_month >= 1 AND harvest_end_month <= 12),
                expected_yield_kg_per_acre INTEGER,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
            """,
            """
            CREATE TABLE IF NOT EXISTS carts (
                id SERIAL PRIMARY KEY,
                user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
                product_id INTEGER REFERENCES products(id) ON DELETE CASCADE,
                quantity_kg INTEGER NOT NULL CHECK (quantity_kg > 0),
                is_bulk BOOLEAN DEFAULT FALSE,
                added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                UNIQUE(user_id, product_id)
            )
            """,
            """
            CREATE TABLE IF NOT EXISTS orders (
                id SERIAL PRIMARY KEY,
                user_id INTEGER REFERENCES users(id),
                order_number VARCHAR(20) UNIQUE NOT NULL,
                total_amount DECIMAL(10,2) NOT NULL,
                subtotal DECIMAL(10,2) NOT NULL,
                shipping_fee DECIMAL(10,2) DEFAULT 0,
                tax DECIMAL(10,2) DEFAULT 0,
                discount DECIMAL(10,2) DEFAULT 0,
                status VARCHAR(20) DEFAULT 'pending',
                payment_method VARCHAR(20),
                payment_status VARCHAR(20) DEFAULT 'unpaid',
                transaction_id VARCHAR(100),
                shipping_address TEXT NOT NULL,
                shipping_tracking VARCHAR(100),
                expected_delivery_date DATE,
                delivered_at TIMESTAMP,
                notes TEXT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
            """,
            """
            CREATE TABLE IF NOT EXISTS order_items (
                id SERIAL PRIMARY KEY,
                order_id INTEGER REFERENCES orders(id) ON DELETE CASCADE,
                product_id INTEGER REFERENCES products(id),
                product_name VARCHAR(200) NOT NULL,
                quantity_kg INTEGER NOT NULL CHECK (quantity_kg > 0),
                is_bulk BOOLEAN DEFAULT FALSE,
                price_per_kg DECIMAL(10,2) NOT NULL,
                total DECIMAL(10,2) NOT NULL,
                harvest_date DATE,
                grade VARCHAR(20)
            )
            """,
            """
            CREATE TABLE IF NOT EXISTS payments (
                id SERIAL PRIMARY KEY,
                order_id INTEGER REFERENCES orders(id),
                amount DECIMAL(10,2) NOT NULL,
                method VARCHAR(20) NOT NULL,
                transaction_id VARCHAR(100) UNIQUE,
                status VARCHAR(20) DEFAULT 'pending',
                provider_response TEXT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                completed_at TIMESTAMP
            )
            """,
            """
            CREATE TABLE IF NOT EXISTS push_subscriptions (
                id SERIAL PRIMARY KEY,
                user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
                endpoint TEXT UNIQUE NOT NULL,
                p256dh TEXT NOT NULL,
                auth TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
            """
        ]
        
        for table in tables:
            Database.execute_query(table, commit=True)
        
        # Insert crop categories
        categories = Database.execute_query("SELECT COUNT(*) as count FROM crop_categories", fetch_one=True)
        if categories and categories['count'] == 0:
            crop_categories = [
                ('Cassava', 'Fresh cassava tubers - Manihot esculenta', 'fa-seedling', 'Year-round (Rainy season)', '12-18 months after planting', 7),
                ('Maize', 'Premium quality maize (corn) - Zea mays', 'fa-corn', 'March-June (Major season), September-November (Minor)', '3-4 months after planting', 365),
                ('Potatoes', 'Fresh Irish potatoes - Solanum tuberosum', 'fa-potato', 'October-December', '3-4 months after planting', 30),
                ('Yam', 'Premium yam tubers - Dioscorea spp.', 'fa-mountain', 'March-June', '8-10 months after planting', 90),
                ('Rice', 'High quality rice grains - Oryza sativa', 'fa-wheat', 'March-June', '4-5 months after planting', 365),
                ('Wheat', 'Premium wheat grains - Triticum spp.', 'fa-wheat-alt', 'November-January', '4-5 months after planting', 365)
            ]
            for cat in crop_categories:
                Database.execute_query(
                    """INSERT INTO crop_categories (name, description, icon, growing_season, harvest_season, storage_life_days)
                       VALUES (%s, %s, %s, %s, %s, %s)""",
                    cat, commit=True
                )
        
        # Insert crop seasons
        seasons = Database.execute_query("SELECT COUNT(*) as count FROM crop_seasons", fetch_one=True)
        if seasons and seasons['count'] == 0:
            crop_seasons = [
                (1, 'Southern Ghana', 3, 6, 12, 2, 15000),   # Cassava
                (1, 'Northern Ghana', 4, 7, 1, 3, 12000),     # Cassava
                (2, 'Southern Ghana', 3, 6, 7, 9, 2500),      # Maize
                (2, 'Northern Ghana', 5, 8, 9, 11, 3000),     # Maize
                (3, 'Southern Ghana', 10, 12, 2, 4, 8000),    # Potatoes
                (3, 'Northern Ghana', 11, 1, 3, 5, 7000),     # Potatoes
                (4, 'Southern Ghana', 3, 6, 11, 1, 10000),    # Yam
                (4, 'Northern Ghana', 4, 7, 12, 2, 12000),    # Yam
                (5, 'Southern Ghana', 3, 6, 7, 10, 3000),     # Rice
                (5, 'Northern Ghana', 5, 8, 9, 12, 3500),     # Rice
                (6, 'Northern Ghana', 11, 1, 3, 5, 2000),     # Wheat
            ]
            for season in crop_seasons:
                Database.execute_query(
                    """INSERT INTO crop_seasons (
                        crop_category_id, region, planting_start_month, planting_end_month,
                        harvest_start_month, harvest_end_month, expected_yield_kg_per_acre
                    ) VALUES (%s, %s, %s, %s, %s, %s, %s)""",
                    season, commit=True
                )
        
        # Insert sample crop products
        products = Database.execute_query("SELECT COUNT(*) as count FROM products", fetch_one=True)
        if products and products['count'] == 0:
            import random
            from datetime import datetime, timedelta
            
            crop_products = [
                # Cassava
                ('Fresh Cassava Tubers', 'High quality cassava tubers, great for fufu, gari, and tapioca', 8.00, 1, 5000,
                 'https://images.unsplash.com/photo-1587485550938-153a7731a6c9?w=400', 'CASSAVA-001', 
                 'Afisiafi', 'Southern Ghana', datetime.now().date(), 'Year-round', 'Dec-Feb',
                 365, 'Store in cool dry place', 'Rich in carbohydrates, vitamin C, and folate',
                 'Premium', 65.0, 98.5, True, 100, 6.50, True, True, 0.5),
                 
                ('Organic Cassava', 'Organically grown cassava, pesticide-free', 10.00, 1, 2000,
                 'https://images.unsplash.com/photo-1587485550938-153a7731a6c9?w=400', 'CASSAVA-ORG-001',
                 'Organic Variety', 'Eastern Ghana', datetime.now().date(), 'Year-round', 'Dec-Feb',
                 365, 'Store in cool dry place', 'Rich in carbohydrates, vitamin C, and folate',
                 'Premium', 60.0, 99.0, True, 100, 8.00, True, True, 0.4),
                 
                # Maize
                ('Premium Ghanaian Maize', 'High quality white maize for banku, kenkey, and porridge', 12.00, 2, 8000,
                 'https://images.unsplash.com/photo-1586201375761-83865001e31c?w=400', 'MAIZE-001',
                 'Obatanpa', 'Northern Ghana', datetime.now().date(), 'March-June', 'July-September',
                 110, 'Store in airtight container', 'Rich in carbohydrates, fiber, and minerals',
                 'Grade 1', 12.0, 99.0, True, 200, 10.00, False, True, 0.3),
                 
                ('Yellow Maize', 'Nutritious yellow maize variety', 14.00, 2, 5000,
                 'https://images.unsplash.com/photo-1586201375761-83865001e31c?w=400', 'MAIZE-YELLOW-001',
                 'Yellow Variety', 'Eastern Ghana', datetime.now().date(), 'March-June', 'July-September',
                 110, 'Store in airtight container', 'Rich in beta-carotene and vitamin A',
                 'Grade 1', 11.0, 98.5, True, 200, 11.50, False, True, 0.3),
                 
                # Potatoes
                ('Fresh Irish Potatoes', 'High quality Irish potatoes for cooking and processing', 15.00, 3, 3000,
                 'https://images.unsplash.com/photo-1587485550938-153a7731a6c9?w=400', 'POTATO-001',
                 'Desiree', 'Southern Ghana', datetime.now().date(), 'October-December', 'February-April',
                 90, 'Store in cool dark place', 'Rich in potassium, vitamin C, and fiber',
                 'Grade 1', 78.0, 97.5, True, 50, 12.00, False, False, 0.2),
                 
                ('Organic Potatoes', 'Organically grown potatoes, no pesticides', 18.00, 3, 1500,
                 'https://images.unsplash.com/photo-1587485550938-153a7731a6c9?w=400', 'POTATO-ORG-001',
                 'Organic Variety', 'Southern Ghana', datetime.now().date(), 'October-December', 'February-April',
                 90, 'Store in cool dark place', 'Rich in potassium, vitamin C, and fiber',
                 'Premium', 76.0, 99.0, True, 50, 14.50, True, True, 0.2),
                 
                # Yam
                ('Premium Yam Tubers', 'High quality yam for fufu, boiled yam, and fried yam', 20.00, 4, 4000,
                 'https://images.unsplash.com/photo-1587485550938-153a7731a6c9?w=400', 'YAM-001',
                 'Puna', 'Northern Ghana', datetime.now().date(), 'March-June', 'November-January',
                 270, 'Store in cool dry place', 'Rich in carbohydrates, potassium, and vitamin C',
                 'Premium', 70.0, 99.0, True, 50, 16.00, True, True, 0.4),
                 
                ('Organic Yam', 'Organically grown yam, no chemicals', 22.00, 4, 2000,
                 'https://images.unsplash.com/photo-1587485550938-153a7731a6c9?w=400', 'YAM-ORG-001',
                 'Organic Variety', 'Northern Ghana', datetime.now().date(), 'March-June', 'November-January',
                 270, 'Store in cool dry place', 'Rich in carbohydrates, potassium, and vitamin C',
                 'Premium', 68.0, 99.5, True, 50, 18.00, True, True, 0.3),
                 
                # Rice
                ('Premium Local Rice', 'High quality Ghanaian rice, fragrant and nutritious', 18.00, 5, 6000,
                 'https://images.unsplash.com/photo-1586201375761-83865001e31c?w=400', 'RICE-001',
                 'Jasmine', 'Northern Ghana', datetime.now().date(), 'March-June', 'July-October',
                 120, 'Store in airtight container', 'Rich in carbohydrates, protein, and minerals',
                 'Grade 1', 13.0, 99.5, True, 100, 15.00, False, True, 0.5),
                 
                ('Brown Rice', 'Whole grain brown rice, more nutritious', 20.00, 5, 3000,
                 'https://images.unsplash.com/photo-1586201375761-83865001e31c?w=400', 'RICE-BROWN-001',
                 'Brown Variety', 'Southern Ghana', datetime.now().date(), 'March-June', 'July-October',
                 120, 'Store in airtight container', 'Rich in fiber, vitamins, and minerals',
                 'Grade 1', 12.5, 99.0, True, 100, 17.00, True, True, 0.5),
                 
                # Wheat
                ('Premium Wheat Grains', 'High quality wheat for baking and processing', 16.00, 6, 2000,
                 'https://images.unsplash.com/photo-1586201375761-83865001e31c?w=400', 'WHEAT-001',
                 'Hard Red Winter', 'Northern Ghana', datetime.now().date(), 'November-January', 'March-May',
                 130, 'Store in cool dry place', 'Rich in carbohydrates, protein, and fiber',
                 'Grade 1', 12.0, 99.5, True, 100, 13.50, False, True, 0.4),
                 
                ('Organic Wheat', 'Organically grown wheat, no pesticides', 20.00, 6, 1000,
                 'https://images.unsplash.com/photo-1586201375761-83865001e31c?w=400', 'WHEAT-ORG-001',
                 'Organic Variety', 'Northern Ghana', datetime.now().date(), 'November-January', 'March-May',
                 130, 'Store in cool dry place', 'Rich in carbohydrates, protein, and fiber',
                 'Premium', 11.5, 99.5, True, 100, 17.00, True, True, 0.3)
            ]
            
            for product in crop_products:
                Database.execute_query(
                    """INSERT INTO products (
                        name, description, price_per_kg, category_id, stock_kg, image_url, sku,
                        crop_variety, growing_region, harvest_date, planting_season, harvest_season,
                        days_to_maturity, storage_instructions, nutritional_info,
                        grade, moisture_content, purity_percentage,
                        is_bulk, bulk_min_kg, bulk_price_per_kg,
                        is_organic, is_fair_trade, carbon_footprint_kg
                    ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)""",
                    product, commit=True
                )
        
        # Create admin user
        admin = Database.execute_query(
            "SELECT id FROM users WHERE email = 'admin@yourstore.com'",
            fetch_one=True
        )
        if not admin:
            hashed = bcrypt.hashpw('admin123'.encode('utf-8'), bcrypt.gensalt()).decode('utf-8')
            Database.execute_query(
                "INSERT INTO users (email, password_hash, full_name, phone, is_admin) VALUES (%s, %s, %s, %s, %s)",
                ('admin@yourstore.com', hashed, 'Admin', '+233200000000', True),
                commit=True
            )

# ==================== UTILITY FUNCTIONS ====================
def hash_password(password):
    salt = bcrypt.gensalt()
    return bcrypt.hashpw(password.encode('utf-8'), salt).decode('utf-8')

def verify_password(password, password_hash):
    return bcrypt.checkpw(password.encode('utf-8'), password_hash.encode('utf-8'))

def generate_jwt(user_id):
    payload = {
        'user_id': user_id,
        'exp': datetime.utcnow() + timedelta(seconds=JWT_EXPIRATION),
        'iat': datetime.utcnow()
    }
    return jwt.encode(payload, SECRET_KEY, algorithm='HS256')

def decode_jwt(token):
    try:
        return jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
    except:
        return None

def token_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.headers.get('Authorization')
        if not token:
            return jsonify({'error': 'Authentication required'}), 401
        try:
            token = token.split(' ')[1]
            payload = decode_jwt(token)
            if not payload:
                return jsonify({'error': 'Invalid or expired token'}), 401
            current_user_id = payload['user_id']
        except:
            return jsonify({'error': 'Invalid token'}), 401
        return f(current_user_id, *args, **kwargs)
    return decorated

def admin_required(f):
    @wraps(f)
    def decorated(current_user_id, *args, **kwargs):
        user = Database.execute_query(
            "SELECT is_admin FROM users WHERE id = %s",
            (current_user_id,),
            fetch_one=True
        )
        if not user or not user['is_admin']:
            return jsonify({'error': 'Admin access required'}), 403
        return f(current_user_id, *args, **kwargs)
    return decorated

def validate_email(email):
    return re.match(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$', email)

def validate_phone(phone):
    return re.match(r'^(\+233|0)[0-9]{9}$', phone)

def generate_order_number():
    timestamp = datetime.now().strftime('%Y%m%d%H%M%S')
    random_part = secrets.token_hex(4).upper()
    return f"CROP-{timestamp}-{random_part}"

def get_cart_key(user_id):
    return f"cart:{user_id}"

def get_current_season(crop_category_id, region=None):
    """Get current growing season for a crop"""
    current_month = datetime.now().month
    query = """SELECT * FROM crop_seasons 
               WHERE crop_category_id = %s AND %s BETWEEN planting_start_month AND planting_end_month"""
    params = [crop_category_id, current_month]
    
    if region:
        query += " AND region = %s"
        params.append(region)
    
    season = Database.execute_query(query, params, fetch_one=True)
    return season

def is_crop_in_season(product_id):
    """Check if a crop product is currently in season"""
    product = Database.execute_query(
        "SELECT category_id, growing_region FROM products WHERE id = %s",
        (product_id,),
        fetch_one=True
    )
    if not product:
        return False
    
    season = get_current_season(product['category_id'], product['growing_region'])
    return season is not None

def get_harvest_status(product):
    """Get harvest status for crop products"""
    if not product.get('harvest_date'):
        return {'status': 'N/A', 'days_until_harvest': None}
    
    harvest_date = product['harvest_date']
    today = datetime.now().date()
    
    if isinstance(harvest_date, str):
        harvest_date = datetime.strptime(harvest_date, '%Y-%m-%d').date()
    
    days_until = (harvest_date - today).days
    
    if days_until < 0:
        return {'status': 'Harvested', 'days_until_harvest': days_until}
    elif days_until < 30:
        return {'status': 'Ready for Harvest', 'days_until_harvest': days_until}
    elif days_until < 60:
        return {'status': 'Almost Ready', 'days_until_harvest': days_until}
    else:
        return {'status': 'Growing', 'days_until_harvest': days_until}

# ==================== AUTH ROUTES ====================
@app.route('/api/auth/signup', methods=['POST'])
def signup():
    data = request.json
    required = ['email', 'password', 'full_name', 'phone']
    
    for field in required:
        if not data.get(field):
            return jsonify({'error': f'{field} is required'}), 400
    
    if not validate_email(data['email']):
        return jsonify({'error': 'Invalid email format'}), 400
    
    if not validate_phone(data['phone']):
        return jsonify({'error': 'Invalid phone. Use +233XXXXXXXXX or 0XXXXXXXXX'}), 400
    
    if len(data['password']) < 8:
        return jsonify({'error': 'Password must be at least 8 characters'}), 400
    
    existing = Database.execute_query(
        "SELECT id FROM users WHERE email = %s",
        (data['email'],),
        fetch_one=True
    )
    if existing:
        return jsonify({'error': 'Email already registered'}), 409
    
    hashed_password = hash_password(data['password'])
    user = Database.execute_query(
        """INSERT INTO users (email, password_hash, full_name, phone, address)
           VALUES (%s, %s, %s, %s, %s) RETURNING id, email, full_name, phone""",
        (data['email'], hashed_password, data['full_name'], data['phone'], data.get('address')),
        fetch_one=True,
        commit=True
    )
    
    return jsonify({'message': 'User created successfully', 'user': user}), 201

@app.route('/api/auth/login', methods=['POST'])
def login():
    data = request.json
    if not data.get('email') or not data.get('password'):
        return jsonify({'error': 'Email and password required'}), 400
    
    user = Database.execute_query(
        "SELECT id, email, password_hash, full_name, phone, is_admin FROM users WHERE email = %s",
        (data['email'],),
        fetch_one=True
    )
    
    if not user or not verify_password(data['password'], user['password_hash']):
        return jsonify({'error': 'Invalid email or password'}), 401
    
    token = generate_jwt(user['id'])
    
    return jsonify({
        'token': token,
        'user': {
            'id': user['id'],
            'email': user['email'],
            'full_name': user['full_name'],
            'phone': user['phone'],
            'is_admin': user['is_admin']
        }
    })

@app.route('/api/auth/me', methods=['GET'])
@token_required
def get_profile(current_user_id):
    user = Database.execute_query(
        "SELECT id, email, full_name, phone, address, is_admin, email_verified, created_at FROM users WHERE id = %s",
        (current_user_id,),
        fetch_one=True
    )
    return jsonify(user)

# ==================== PRODUCT ROUTES ====================
@app.route('/api/products', methods=['GET'])
def get_products():
    category = request.args.get('category')
    search = request.args.get('search')
    min_price = request.args.get('min_price')
    max_price = request.args.get('max_price')
    sort_by = request.args.get('sort_by', 'created_at')
    order = request.args.get('order', 'DESC')
    limit = int(request.args.get('limit', 20))
    offset = int(request.args.get('offset', 0))
    is_organic = request.args.get('is_organic')
    is_bulk = request.args.get('is_bulk')
    in_season = request.args.get('in_season')
    grade = request.args.get('grade')
    
    query = """
        SELECT p.*, c.name as category_name, c.icon, c.growing_season, c.harvest_season
        FROM products p
        LEFT JOIN crop_categories c ON p.category_id = c.id
        WHERE p.is_active = TRUE AND p.stock_kg > 0
    """
    params = []
    
    if category:
        query += " AND (p.category_id = %s OR c.name ILIKE %s)"
        params.extend([category, f"%{category}%"])
    
    if search:
        query += " AND (p.name ILIKE %s OR p.crop_variety ILIKE %s OR c.name ILIKE %s)"
        params.extend([f"%{search}%", f"%{search}%", f"%{search}%"])
    
    if min_price:
        query += " AND p.price_per_kg >= %s"
        params.append(float(min_price))
    
    if max_price:
        query += " AND p.price_per_kg <= %s"
        params.append(float(max_price))
    
    if is_organic == 'true':
        query += " AND p.is_organic = TRUE"
    
    if is_bulk == 'true':
        query += " AND p.is_bulk = TRUE"
    
    if grade:
        query += " AND p.grade = %s"
        params.append(grade)
    
    if in_season == 'true':
        current_month = datetime.now().month
        query += """ AND EXISTS (
            SELECT 1 FROM crop_seasons cs 
            WHERE cs.crop_category_id = p.category_id 
            AND %s BETWEEN cs.planting_start_month AND cs.planting_end_month
            AND cs.region = p.growing_region
        )"""
        params.append(current_month)
    
    allowed_sort = ['price_per_kg', 'name', 'created_at', 'stock_kg', 'harvest_date']
    if sort_by not in allowed_sort:
        sort_by = 'created_at'
    order = 'DESC' if order.upper() == 'DESC' else 'ASC'
    
    query += f" ORDER BY p.{sort_by} {order} LIMIT %s OFFSET %s"
    params.extend([limit, offset])
    
    products = Database.execute_query(query, params, fetch_all=True)
    
    # Add season and harvest status
    for product in products:
        product['in_season'] = is_crop_in_season(product['id'])
        product['harvest_status'] = get_harvest_status(product)
        product['bulk_discount'] = None
        if product.get('is_bulk') and product.get('bulk_price_per_kg'):
            discount = ((product['price_per_kg'] - product['bulk_price_per_kg']) / product['price_per_kg']) * 100
            product['bulk_discount'] = round(discount, 1)
    
    count_query = "SELECT COUNT(*) FROM products p WHERE p.is_active = TRUE"
    total = Database.execute_query(count_query, fetch_one=True)['count']
    
    return jsonify({
        'products': products,
        'pagination': {
            'total': total,
            'limit': limit,
            'offset': offset,
            'pages': (total + limit - 1) // limit
        }
    })

@app.route('/api/products/<int:product_id>', methods=['GET'])
def get_product(product_id):
    product = Database.execute_query(
        """SELECT p.*, c.name as category_name, c.icon, c.growing_season, c.harvest_season
           FROM products p
           LEFT JOIN crop_categories c ON p.category_id = c.id
           WHERE p.id = %s AND p.is_active = TRUE""",
        (product_id,),
        fetch_one=True
    )
    if not product:
        return jsonify({'error': 'Product not found'}), 404
    
    product['in_season'] = is_crop_in_season(product['id'])
    product['harvest_status'] = get_harvest_status(product)
    
    return jsonify(product)

@app.route('/api/products/by-crop/<crop_name>', methods=['GET'])
def get_products_by_crop(crop_name):
    """Get products by crop category name"""
    query = """
        SELECT p.*, c.name as category_name
        FROM products p
        JOIN crop_categories c ON p.category_id = c.id
        WHERE c.name ILIKE %s AND p.is_active = TRUE AND p.stock_kg > 0
        ORDER BY p.price_per_kg ASC
    """
    products = Database.execute_query(query, (f"%{crop_name}%",), fetch_all=True)
    
    for product in products:
        product['in_season'] = is_crop_in_season(product['id'])
        product['harvest_status'] = get_harvest_status(product)
    
    return jsonify(products)

@app.route('/api/products/crops', methods=['GET'])
def get_crop_categories():
    categories = Database.execute_query(
        "SELECT * FROM crop_categories ORDER BY name",
        fetch_all=True
    )
    return jsonify(categories)

@app.route('/api/products/seasons/<int:category_id>', methods=['GET'])
def get_crop_seasons(category_id):
    seasons = Database.execute_query(
        "SELECT * FROM crop_seasons WHERE crop_category_id = %s ORDER BY region",
        (category_id,),
        fetch_all=True
    )
    return jsonify(seasons)

@app.route('/api/products', methods=['POST'])
@token_required
@admin_required
def create_product(current_user_id):
    data = request.json
    required = ['name', 'price_per_kg', 'category_id', 'stock_kg']
    
    for field in required:
        if not data.get(field):
            return jsonify({'error': f'{field} is required'}), 400
    
    if data.get('price_per_kg', 0) < 0:
        return jsonify({'error': 'Price must be greater than 0'}), 400
    
    harvest_date = None
    if data.get('harvest_date'):
        harvest_date = datetime.strptime(data['harvest_date'], '%Y-%m-%d').date()
    
    product = Database.execute_query(
        """INSERT INTO products (
            name, description, price_per_kg, category_id, stock_kg, image_url, sku,
            crop_variety, growing_region, harvest_date, planting_season, harvest_season,
            days_to_maturity, storage_instructions, nutritional_info,
            grade, moisture_content, purity_percentage,
            is_bulk, bulk_min_kg, bulk_price_per_kg,
            is_organic, is_fair_trade, carbon_footprint_kg
        ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s) 
        RETURNING id, name, price_per_kg, stock_kg""",
        (
            data['name'],
            data.get('description'),
            data['price_per_kg'],
            data['category_id'],
            data['stock_kg'],
            data.get('image_url'),
            data.get('sku'),
            data.get('crop_variety'),
            data.get('growing_region'),
            harvest_date,
            data.get('planting_season'),
            data.get('harvest_season'),
            data.get('days_to_maturity'),
            data.get('storage_instructions'),
            data.get('nutritional_info'),
            data.get('grade'),
            data.get('moisture_content'),
            data.get('purity_percentage'),
            data.get('is_bulk', False),
            data.get('bulk_min_kg'),
            data.get('bulk_price_per_kg'),
            data.get('is_organic', False),
            data.get('is_fair_trade', False),
            data.get('carbon_footprint_kg')
        ),
        fetch_one=True,
        commit=True
    )
    
    return jsonify(product), 201

@app.route('/api/products/<int:product_id>', methods=['PUT'])
@token_required
@admin_required
def update_product(current_user_id, product_id):
    data = request.json
    allowed = [
        'name', 'description', 'price_per_kg', 'category_id', 'stock_kg', 'image_url',
        'sku', 'crop_variety', 'growing_region', 'harvest_date', 'planting_season',
        'harvest_season', 'days_to_maturity', 'storage_instructions', 'nutritional_info',
        'grade', 'moisture_content', 'purity_percentage', 'is_active',
        'is_bulk', 'bulk_min_kg', 'bulk_price_per_kg', 'is_organic', 'is_fair_trade',
        'carbon_footprint_kg'
    ]
    updates = []
    params = []
    
    for field in allowed:
        if field in data:
            if field == 'price_per_kg' and data['price_per_kg'] < 0:
                return jsonify({'error': 'Price must be greater than 0'}), 400
            if field == 'harvest_date' and data[field]:
                data[field] = datetime.strptime(data[field], '%Y-%m-%d').date()
            updates.append(f"{field} = %s")
            params.append(data[field])
    
    if not updates:
        return jsonify({'error': 'No fields to update'}), 400
    
    updates.append("updated_at = CURRENT_TIMESTAMP")
    params.append(product_id)
    
    product = Database.execute_query(
        f"UPDATE products SET {', '.join(updates)} WHERE id = %s RETURNING *",
        params,
        fetch_one=True,
        commit=True
    )
    
    if not product:
        return jsonify({'error': 'Product not found'}), 404
    
    return jsonify(product)

@app.route('/api/products/<int:product_id>', methods=['DELETE'])
@token_required
@admin_required
def delete_product(current_user_id, product_id):
    product = Database.execute_query(
        "DELETE FROM products WHERE id = %s RETURNING id",
        (product_id,),
        fetch_one=True,
        commit=True
    )
    if not product:
        return jsonify({'error': 'Product not found'}), 404
    return jsonify({'message': 'Product deleted successfully'})

# ==================== CART ROUTES ====================
@app.route('/api/cart', methods=['GET'])
@token_required
def get_cart(current_user_id):
    cart_key = get_cart_key(current_user_id)
    
    if redis_client:
        cached = redis_client.get(cart_key)
        if cached:
            return jsonify(json.loads(cached))
    
    cart_items = Database.execute_query(
        """SELECT c.*, p.name, p.price_per_kg, p.image_url, p.stock_kg,
           p.is_bulk, p.bulk_price_per_kg, p.grade, p.harvest_date
           FROM carts c
           JOIN products p ON c.product_id = p.id
           WHERE c.user_id = %s""",
        (current_user_id,),
        fetch_all=True
    )
    
    for item in cart_items:
        price = item['bulk_price_per_kg'] if item['is_bulk'] and item['bulk_price_per_kg'] else item['price_per_kg']
        item['effective_price'] = price
    
    cart = {
        'items': cart_items,
        'total_items': sum(item['quantity_kg'] for item in cart_items),
        'subtotal': sum(item['effective_price'] * item['quantity_kg'] for item in cart_items)
    }
    
    if redis_client:
        redis_client.setex(cart_key, 300, json.dumps(cart))
    
    return jsonify(cart)

@app.route('/api/cart/add', methods=['POST'])
@token_required
def add_to_cart(current_user_id):
    data = request.json
    product_id = data.get('product_id')
    quantity_kg = data.get('quantity_kg', 1)
    is_bulk = data.get('is_bulk', False)
    
    if not product_id:
        return jsonify({'error': 'Product ID required'}), 400
    
    if quantity_kg <= 0:
        return jsonify({'error': 'Quantity must be greater than 0'}), 400
    
    product = Database.execute_query(
        "SELECT id, name, price_per_kg, stock_kg, is_bulk, bulk_min_kg, bulk_price_per_kg FROM products WHERE id = %s AND is_active = TRUE",
        (product_id,),
        fetch_one=True
    )
    
    if not product:
        return jsonify({'error': 'Product not found'}), 404
    
    if is_bulk and product['is_bulk'] and quantity_kg < product['bulk_min_kg']:
        return jsonify({'error': f'Bulk minimum is {product["bulk_min_kg"]} kg'}), 400
    
    if product['stock_kg'] < quantity_kg:
        return jsonify({'error': f'Only {product["stock_kg"]} kg in stock'}), 400
    
    existing = Database.execute_query(
        "SELECT id, quantity_kg FROM carts WHERE user_id = %s AND product_id = %s",
        (current_user_id, product_id),
        fetch_one=True
    )
    
    if existing:
        new_quantity = existing['quantity_kg'] + quantity_kg
        Database.execute_query(
            "UPDATE carts SET quantity_kg = %s, is_bulk = %s WHERE id = %s",
            (new_quantity, is_bulk, existing['id']),
            commit=True
        )
    else:
        Database.execute_query(
            "INSERT INTO carts (user_id, product_id, quantity_kg, is_bulk) VALUES (%s, %s, %s, %s)",
            (current_user_id, product_id, quantity_kg, is_bulk),
            commit=True
        )
    
    if redis_client:
        redis_client.delete(get_cart_key(current_user_id))
    
    return jsonify({'message': 'Product added to cart'})

@app.route('/api/cart/update', methods=['PUT'])
@token_required
def update_cart(current_user_id):
    data = request.json
    product_id = data.get('product_id')
    quantity_kg = data.get('quantity_kg')
    
    if not product_id or quantity_kg is None:
        return jsonify({'error': 'Product ID and quantity required'}), 400
    
    if quantity_kg < 0:
        return jsonify({'error': 'Quantity cannot be negative'}), 400
    
    if quantity_kg == 0:
        Database.execute_query(
            "DELETE FROM carts WHERE user_id = %s AND product_id = %s",
            (current_user_id, product_id),
            commit=True
        )
    else:
        product = Database.execute_query(
            "SELECT stock_kg FROM products WHERE id = %s",
            (product_id,),
            fetch_one=True
        )
        if not product:
            return jsonify({'error': 'Product not found'}), 404
        if product['stock_kg'] < quantity_kg:
            return jsonify({'error': f'Only {product["stock_kg"]} kg in stock'}), 400
        
        Database.execute_query(
            "UPDATE carts SET quantity_kg = %s WHERE user_id = %s AND product_id = %s",
            (quantity_kg, current_user_id, product_id),
            commit=True
        )
    
    if redis_client:
        redis_client.delete(get_cart_key(current_user_id))
    
    return jsonify({'message': 'Cart updated'})

@app.route('/api/cart/clear', methods=['DELETE'])
@token_required
def clear_cart(current_user_id):
    Database.execute_query(
        "DELETE FROM carts WHERE user_id = %s",
        (current_user_id,),
        commit=True
    )
    if redis_client:
        redis_client.delete(get_cart_key(current_user_id))
    return jsonify({'message': 'Cart cleared'})

# ==================== ORDER ROUTES ====================
@app.route('/api/orders', methods=['GET'])
@token_required
def get_orders(current_user_id):
    status = request.args.get('status')
    query = "SELECT * FROM orders WHERE user_id = %s"
    params = [current_user_id]
    
    if status:
        query += " AND status = %s"
        params.append(status)
    
    query += " ORDER BY created_at DESC"
    orders = Database.execute_query(query, params, fetch_all=True)
    
    for order in orders:
        items = Database.execute_query(
            "SELECT * FROM order_items WHERE order_id = %s",
            (order['id'],),
            fetch_all=True
        )
        order['items'] = items
    
    return jsonify(orders)

@app.route('/api/orders/<int:order_id>', methods=['GET'])
@token_required
def get_order(current_user_id, order_id):
    order = Database.execute_query(
        "SELECT * FROM orders WHERE id = %s AND user_id = %s",
        (order_id, current_user_id),
        fetch_one=True
    )
    if not order:
        return jsonify({'error': 'Order not found'}), 404
    
    items = Database.execute_query(
        "SELECT * FROM order_items WHERE order_id = %s",
        (order_id,),
        fetch_all=True
    )
    order['items'] = items
    return jsonify(order)

@app.route('/api/orders', methods=['POST'])
@token_required
def create_order(current_user_id):
    data = request.json
    address = data.get('shipping_address')
    notes = data.get('notes')
    
    if not address:
        return jsonify({'error': 'Shipping address required'}), 400
    
    cart_items = Database.execute_query(
        """SELECT c.*, p.name, p.price_per_kg, p.stock_kg, p.is_bulk, p.bulk_price_per_kg,
           p.grade, p.harvest_date
           FROM carts c
           JOIN products p ON c.product_id = p.id
           WHERE c.user_id = %s""",
        (current_user_id,),
        fetch_all=True
    )
    
    if not cart_items:
        return jsonify({'error': 'Cart is empty'}), 400
    
    subtotal = 0
    order_items = []
    expected_delivery = datetime.now().date() + timedelta(days=3)
    
    for item in cart_items:
        if item['stock_kg'] < item['quantity_kg']:
            return jsonify({'error': f'Insufficient stock for {item["name"]}'}), 400
        
        price = item['bulk_price_per_kg'] if item['is_bulk'] and item['bulk_price_per_kg'] else item['price_per_kg']
        total = price * item['quantity_kg']
        subtotal += total
        
        order_items.append({
            'product_id': item['product_id'],
            'product_name': item['name'],
            'quantity_kg': item['quantity_kg'],
            'is_bulk': item['is_bulk'],
            'price_per_kg': price,
            'total': total,
            'grade': item.get('grade'),
            'harvest_date': item.get('harvest_date')
        })
    
    shipping_fee = 50.00 if subtotal < 500 else 0
    tax = subtotal * 0.125
    total_amount = subtotal + shipping_fee + tax
    
    order_number = generate_order_number()
    order = Database.execute_query(
        """INSERT INTO orders (
            user_id, order_number, total_amount, subtotal, shipping_fee, tax, 
            shipping_address, notes, status, expected_delivery_date
        ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, 'pending', %s)
        RETURNING *""",
        (
            current_user_id, order_number, total_amount, subtotal, shipping_fee, tax,
            address, notes, expected_delivery
        ),
        fetch_one=True,
        commit=True
    )
    
    for item in order_items:
        Database.execute_query(
            """INSERT INTO order_items (
                order_id, product_id, product_name, quantity_kg, is_bulk, price_per_kg, total,
                grade, harvest_date
            ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)""",
            (
                order['id'], item['product_id'], item['product_name'],
                item['quantity_kg'], item['is_bulk'], item['price_per_kg'], item['total'],
                item['grade'], item['harvest_date']
            ),
            commit=True
        )
        Database.execute_query(
            "UPDATE products SET stock_kg = stock_kg - %s WHERE id = %s",
            (item['quantity_kg'], item['product_id']),
            commit=True
        )
    
    Database.execute_query(
        "DELETE FROM carts WHERE user_id = %s",
        (current_user_id,),
        commit=True
    )
    
    if redis_client:
        redis_client.delete(get_cart_key(current_user_id))
    
    # Send WhatsApp notification
    send_whatsapp_order_confirmation(current_user_id, order['id'])
    
    return jsonify(order), 201

@app.route('/api/orders/<int:order_id>/status', methods=['PUT'])
@token_required
def update_order_status(current_user_id, order_id):
    data = request.json
    status = data.get('status')
    
    if not status:
        return jsonify({'error': 'Status required'}), 400
    
    valid = ['pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled']
    if status not in valid:
        return jsonify({'error': f'Invalid status. Must be one of {valid}'}), 400
    
    user = Database.execute_query(
        "SELECT is_admin FROM users WHERE id = %s",
        (current_user_id,),
        fetch_one=True
    )
    
    if not user['is_admin']:
        order = Database.execute_query(
            "SELECT id FROM orders WHERE id = %s AND user_id = %s",
            (order_id, current_user_id),
            fetch_one=True
        )
        if not order:
            return jsonify({'error': 'Order not found'}), 404
        if status != 'cancelled':
            return jsonify({'error': 'Only admins can change order status'}), 403
    
    updates = ["status = %s"]
    params = [status]
    
    if status == 'delivered':
        updates.append("delivered_at = CURRENT_TIMESTAMP")
    
    params.append(order_id)
    
    order = Database.execute_query(
        f"UPDATE orders SET {', '.join(updates)} WHERE id = %s RETURNING *",
        params,
        fetch_one=True,
        commit=True
    )
    
    if not order:
        return jsonify({'error': 'Order not found'}), 404
    
    send_whatsapp_status_update(order['user_id'], order['id'], status)
    return jsonify(order)

# ==================== PAYMENT ROUTES ====================
@app.route('/api/payments/initiate-momo', methods=['POST'])
@token_required
def initiate_momo(current_user_id):
    data = request.json
    order_id = data.get('order_id')
    phone = data.get('phone')
    
    if not order_id or not phone:
        return jsonify({'error': 'Order ID and phone required'}), 400
    
    order = Database.execute_query(
        "SELECT * FROM orders WHERE id = %s AND user_id = %s",
        (order_id, current_user_id),
        fetch_one=True
    )
    
    if not order:
        return jsonify({'error': 'Order not found'}), 404
    
    if order['payment_status'] == 'paid':
        return jsonify({'error': 'Order already paid'}), 400
    
    if phone.startswith('0'):
        phone = '233' + phone[1:]
    elif not phone.startswith('233'):
        phone = '233' + phone
    
    transaction_id = f"MOMO-{secrets.token_hex(8)}"
    
    Database.execute_query(
        """INSERT INTO payments (order_id, amount, method, transaction_id, status)
           VALUES (%s, %s, %s, %s, 'pending')""",
        (order_id, order['total_amount'], 'momo', transaction_id),
        commit=True
    )
    
    # For demo, auto-complete payment
    Database.execute_query(
        """UPDATE orders SET 
           payment_status = 'paid', 
           payment_method = 'momo',
           transaction_id = %s,
           status = 'confirmed'
           WHERE id = %s""",
        (transaction_id, order_id),
        commit=True
    )
    
    Database.execute_query(
        """UPDATE payments SET 
           status = 'completed',
           completed_at = CURRENT_TIMESTAMP
           WHERE transaction_id = %s""",
        (transaction_id,),
        commit=True
    )
    
    send_whatsapp_payment_confirmation(current_user_id, order_id)
    
    return jsonify({
        'message': 'Payment successful',
        'transaction_id': transaction_id
    })

@app.route('/api/payment/callback', methods=['POST'])
def payment_callback():
    data = request.json
    transaction_id = data.get('TransactionId')
    status = data.get('Status')
    
    if status == 'Success':
        order_id = data.get('TransactionId')
        Database.execute_query(
            "UPDATE orders SET payment_status = 'paid', status = 'confirmed' WHERE id = %s",
            (order_id,),
            commit=True
        )
        Database.execute_query(
            "UPDATE payments SET status = 'completed', completed_at = CURRENT_TIMESTAMP WHERE transaction_id = %s",
            (transaction_id,),
            commit=True
        )
    
    return jsonify({'status': 'ok'}), 200

# ==================== ADMIN ROUTES ====================
@app.route('/api/admin/dashboard', methods=['GET'])
@token_required
@admin_required
def admin_dashboard(current_user_id):
    today = datetime.now().date()
    
    revenue_today = Database.execute_query(
        "SELECT COALESCE(SUM(total_amount), 0) as total FROM orders WHERE DATE(created_at) = %s AND status IN ('paid', 'confirmed', 'shipped', 'delivered')",
        (today,),
        fetch_one=True
    )
    
    total_orders = Database.execute_query("SELECT COUNT(*) as count FROM orders", fetch_one=True)
    pending_orders = Database.execute_query("SELECT COUNT(*) as count FROM orders WHERE status = 'pending'", fetch_one=True)
    total_products = Database.execute_query("SELECT COUNT(*) as count FROM products", fetch_one=True)
    total_users = Database.execute_query("SELECT COUNT(*) as count FROM users WHERE is_admin = FALSE", fetch_one=True)
    
    # Crop specific stats
    total_stock_kg = Database.execute_query("SELECT COALESCE(SUM(stock_kg), 0) as total FROM products", fetch_one=True)
    organic_products = Database.execute_query("SELECT COUNT(*) as count FROM products WHERE is_organic = TRUE", fetch_one=True)
    bulk_products = Database.execute_query("SELECT COUNT(*) as count FROM products WHERE is_bulk = TRUE", fetch_one=True)
    
    low_stock = Database.execute_query(
        "SELECT id, name, stock_kg, image_url, grade FROM products WHERE stock_kg < 100 ORDER BY stock_kg ASC",
        fetch_all=True
    )
    
    recent_orders = Database.execute_query(
        """SELECT o.*, u.full_name, u.email 
           FROM orders o
           JOIN users u ON o.user_id = u.id
           ORDER BY o.created_at DESC LIMIT 10""",
        fetch_all=True
    )
    
    revenue_7_days = Database.execute_query(
        """SELECT DATE(created_at) as date, SUM(total_amount) as total
           FROM orders
           WHERE created_at > NOW() - INTERVAL '7 days' AND status IN ('paid', 'confirmed', 'shipped', 'delivered')
           GROUP BY DATE(created_at)
           ORDER BY DATE(created_at)""",
        fetch_all=True
    )
    
    top_products = Database.execute_query(
        """SELECT p.id, p.name, p.image_url, SUM(oi.quantity_kg) as total_sold_kg, SUM(oi.total) as revenue
           FROM order_items oi
           JOIN products p ON oi.product_id = p.id
           JOIN orders o ON oi.order_id = o.id
           WHERE o.status IN ('paid', 'confirmed', 'shipped', 'delivered')
           GROUP BY p.id, p.name, p.image_url
           ORDER BY total_sold_kg DESC LIMIT 10""",
        fetch_all=True
    )
    
    # Crop category breakdown
    crop_breakdown = Database.execute_query(
        """SELECT c.name, COUNT(p.id) as product_count, SUM(p.stock_kg) as total_stock_kg
           FROM crop_categories c
           LEFT JOIN products p ON c.id = p.category_id
           GROUP BY c.id, c.name
           ORDER BY total_stock_kg DESC""",
        fetch_all=True
    )
    
    return jsonify({
        'stats': {
            'today_revenue': float(revenue_today['total']),
            'total_orders': total_orders['count'],
            'pending_orders': pending_orders['count'],
            'total_products': total_products['count'],
            'total_users': total_users['count'],
            'total_stock_kg': float(total_stock_kg['total']),
            'organic_products': organic_products['count'],
            'bulk_products': bulk_products['count']
        },
        'low_stock': low_stock,
        'recent_orders': recent_orders,
        'revenue_last_7_days': revenue_7_days,
        'top_products': top_products,
        'crop_breakdown': crop_breakdown
    })

@app.route('/api/admin/orders', methods=['GET'])
@token_required
@admin_required
def admin_orders(current_user_id):
    status = request.args.get('status')
    query = """SELECT o.*, u.full_name, u.email, u.phone 
               FROM orders o
               JOIN users u ON o.user_id = u.id"""
    params = []
    
    if status:
        query += " WHERE o.status = %s"
        params.append(status)
    
    query += " ORDER BY o.created_at DESC"
    orders = Database.execute_query(query, params, fetch_all=True)
    return jsonify(orders)

@app.route('/api/admin/users', methods=['GET'])
@token_required
@admin_required
def admin_users(current_user_id):
    users = Database.execute_query(
        "SELECT id, email, full_name, phone, address, is_admin, email_verified, created_at FROM users ORDER BY created_at DESC",
        fetch_all=True
    )
    return jsonify(users)

@app.route('/api/admin/users/<int:user_id>', methods=['DELETE'])
@token_required
@admin_required
def delete_user(current_user_id, user_id):
    if user_id == current_user_id:
        return jsonify({'error': 'Cannot delete your own account'}), 400
    user = Database.execute_query(
        "DELETE FROM users WHERE id = %s RETURNING id",
        (user_id,),
        fetch_one=True,
        commit=True
    )
    if not user:
        return jsonify({'error': 'User not found'}), 404
    return jsonify({'message': 'User deleted successfully'})

# ==================== WHATSAPP ROUTES ====================
def send_whatsapp_message(phone, message):
    if not TWILIO_ACCOUNT_SID or not TWILIO_AUTH_TOKEN:
        print(f"WhatsApp: {message}")
        return None
    
    try:
        client = Client(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN)
        msg = client.messages.create(
            from_=f'whatsapp:{TWILIO_WHATSAPP_NUMBER}',
            body=message,
            to=f'whatsapp:{phone}'
        )
        return msg.sid
    except Exception as e:
        print(f"WhatsApp error: {e}")
        return None

def send_whatsapp_order_confirmation(user_id, order_id):
    user = Database.execute_query(
        "SELECT phone, full_name FROM users WHERE id = %s",
        (user_id,),
        fetch_one=True
    )
    order = Database.execute_query("SELECT * FROM orders WHERE id = %s", (order_id,), fetch_one=True)
    items = Database.execute_query("SELECT * FROM order_items WHERE order_id = %s", (order_id,), fetch_all=True)
    
    if not user:
        return
    
    message = f"🌾 *Crop Order Confirmation*\n\nHello {user['full_name']}! 👋\n"
    message += f"Your crop order #{order['order_number']} has been received.\n\n*Order Details:*\n"
    
    for item in items:
        message += f"• {item['product_name']} x{item['quantity_kg']}kg = ₵{float(item['total']):.2f}\n"
        if item.get('grade'):
            message += f"  Grade: {item['grade']}\n"
    
    message += f"\n*Subtotal:* ₵{float(order['subtotal']):.2f}\n"
    message += f"*Shipping:* ₵{float(order['shipping_fee']):.2f}\n"
    message += f"*Tax:* ₵{float(order['tax']):.2f}\n"
    message += f"*Total:* ₵{float(order['total_amount']):.2f}\n\n"
    message += f"📍 *Shipping to:* {order['shipping_address']}\n"
    if order['expected_delivery_date']:
        message += f"📅 *Expected Delivery:* {order['expected_delivery_date']}\n\n"
    message += "We'll notify you when your crop order is processed.\n\n"
    message += "Thank you for supporting Ghanaian farmers! 🌱"
    
    send_whatsapp_message(user['phone'], message)

def send_whatsapp_status_update(user_id, order_id, status):
    user = Database.execute_query(
        "SELECT phone, full_name FROM users WHERE id = %s",
        (user_id,),
        fetch_one=True
    )
    order = Database.execute_query("SELECT * FROM orders WHERE id = %s", (order_id,), fetch_one=True)
    
    if not user:
        return
    
    status_messages = {
        'confirmed': '✅ Your crop order has been confirmed!',
        'processing': '🔧 We are preparing your crop order.',
        'shipped': '🚚 Your crop order has been shipped!',
        'delivered': '📦 Your crop order has been delivered. Thank you!',
        'cancelled': '❌ Your crop order has been cancelled.'
    }
    
    message = f"🌾 *Crop Order Update*\n\nHello {user['full_name']}!\n"
    message += f"Order #{order['order_number']}: {status_messages.get(status, 'Status updated')}\n\n"
    
    if status == 'shipped' and order.get('shipping_tracking'):
        message += f"🔢 *Tracking Number:* {order['shipping_tracking']}\n\n"
    
    message += "Thanks for choosing Ghanaian crops! 🌱"
    
    send_whatsapp_message(user['phone'], message)

def send_whatsapp_payment_confirmation(user_id, order_id):
    user = Database.execute_query(
        "SELECT phone, full_name FROM users WHERE id = %s",
        (user_id,),
        fetch_one=True
    )
    order = Database.execute_query("SELECT * FROM orders WHERE id = %s", (order_id,), fetch_one=True)
    
    if not user:
        return
    
    message = f"💳 *Payment Confirmation*\n\nHello {user['full_name']}!\n"
    message += f"Your payment of ₵{float(order['total_amount']):.2f} for crop order #{order['order_number']} has been confirmed.\n\n"
    message += "We will process your crop order shortly. Thank you for supporting local farmers! 🙏"
    
    send_whatsapp_message(user['phone'], message)

# ==================== CHATBOT ROUTES ====================
CHATBOT_QA = {
    'how to order': "🌾 To place a crop order:\n1. Browse our crop products\n2. Select quantity in kg\n3. Add to cart\n4. Proceed to checkout\n5. Enter shipping address\n6. Select payment method (MTN MoMo, Card, Cash)\n7. Confirm your order",
    'delivery time': "📦 Delivery times:\n• Accra: 2-3 business days\n• Other regions: 5-7 business days\n• Bulk orders: 3-5 business days",
    'delivery fee': "🚚 Shipping fees:\n• Free shipping on orders over ₵500\n• ₵50 for orders under ₵500",
    'return policy': "🔄 Returns:\n• 7-day return policy for crop products\n• Items must be in original condition\n• Contact support to initiate return\n• Refund processed within 3-5 business days",
    'momo payment': "💰 MTN MoMo payments:\n1. Select MoMo at checkout\n2. Enter your MoMo phone number\n3. Confirm payment on your phone\n4. Receive payment confirmation",
    'contact support': "📞 Customer Support:\n• Email: support@yourstore.com\n• Phone: +233 20 000 0000\n• WhatsApp: +233 20 000 0000",
    'cassava': "🌱 Cassava Info:\n• Growing season: Year-round (rainy season)\n• Harvest: 12-18 months after planting\n• Storage: Up to 7 days fresh\n• Uses: Fufu, gari, tapioca, starch",
    'maize': "🌽 Maize (Corn) Info:\n• Growing season: March-June (Major), Sept-Nov (Minor)\n• Harvest: 3-4 months after planting\n• Storage: Up to 12 months (dried)\n• Uses: Banku, kenkey, porridge, flour",
    'potatoes': "🥔 Potato Info:\n• Growing season: October-December\n• Harvest: 3-4 months after planting\n• Storage: Up to 30 days\n• Uses: Cooking, processing, chips",
    'yam': "🍠 Yam Info:\n• Growing season: March-June\n• Harvest: 8-10 months after planting\n• Storage: Up to 90 days\n• Uses: Fufu, boiled, fried, porridge",
    'rice': "🍚 Rice Info:\n• Growing season: March-June\n• Harvest: 4-5 months after planting\n• Storage: Up to 12 months\n• Uses: Cooking, jollof, plain rice",
    'wheat': "🌾 Wheat Info:\n• Growing season: November-January\n• Harvest: 4-5 months after planting\n• Storage: Up to 12 months\n• Uses: Baking, flour, pasta",
    'bulk': "📦 Bulk Orders:\n• Available on all crops\n• Minimum: 50-100kg depending on crop\n• Discounted prices\n• Great for restaurants, bakeries, and processors",
    'organic': "🌿 Organic Crops:\n• Grown without pesticides\n• Certified organic available\n• Check for organic badge on products",
    'seasonal': "📅 Seasonal Crops:\n• Check if crop is 'In Season'\n• Fresh harvest available\n• Better prices during season"
}

@app.route('/api/chatbot', methods=['POST'])
def chatbot():
    data = request.json
    message = data.get('message', '').lower()
    
    # Check for crop-specific queries
    crop_keywords = ['cassava', 'maize', 'potato', 'yam', 'rice', 'wheat', 'corn']
    for crop in crop_keywords:
        if crop in message:
            for key in CHATBOT_QA:
                if crop in key or key in crop:
                    return jsonify({
                        'response': CHATBOT_QA[key],
                        'suggestions': ['Bulk orders', 'Organic', 'Seasonal', 'How to order']
                    })
    
    for keyword, response in CHATBOT_QA.items():
        if any(word in message for word in keyword.split()):
            return jsonify({
                'response': response,
                'suggestions': ['How to order', 'Delivery time', 'Momo payment', 'Contact support', 'Bulk orders']
            })
    
    return jsonify({
        'response': "🌾 I'm your crop assistant. Please try:\n• 'How to order'\n• 'Delivery time'\n• 'Cassava'\n• 'Maize'\n• 'Potatoes'\n• 'Yam'\n• 'Rice'\n• 'Wheat'\n• 'Bulk orders'\n• 'Organic'\n• 'Seasonal'\n\nOr email us at support@yourstore.com for help!",
        'suggestions': ['How to order', 'Cassava', 'Maize', 'Rice', 'Bulk orders', 'Contact support']
    })

# ==================== HEALTH CHECK ====================
@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'timestamp': datetime.utcnow().isoformat()})

# ==================== SERVE FRONTEND ====================
@app.route('/')
def index():
    return send_from_directory('.', 'index.html')

@app.route('/admin')
def admin():
    return send_from_directory('.', 'admin.html')

# ==================== INITIALIZE ====================
if __name__ == '__main__':
    print("🌾 Initializing Crop Database...")
    Database.init_db()
    print("✅ Database ready!")
    print("🌱 Crops loaded: Cassava, Maize, Potatoes, Yam, Rice, Wheat")
    print("📅 Seasonal data included!")
    print("🚀 Server running on http://localhost:5000")
    app.run(host='0.0.0.0', port=5000, debug=True)
