from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy import func
from functools import wraps
from datetime import datetime

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///site.db'  # Replace this with the actual URI for your database
db = SQLAlchemy(app)

# Define the models
class ProductMeta(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    productName = db.Column(db.String(100), nullable=False)
    onboardInventoryDate = db.Column(db.DateTime, nullable=False)
    sellerID = db.Column(db.Integer, db.ForeignKey('sellerMeta.id'), nullable=False)

class SellerMeta(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    sellerName = db.Column(db.String(100), nullable=False)
    sellerGST = db.Column(db.String(15), nullable=False, unique=True)
    onboardingDate = db.Column(db.DateTime, nullable=False)
    products = db.relationship('ProductMeta', backref='seller', lazy=True)

class InventoryLog(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    addInventoryDate = db.Column(db.DateTime, nullable=False, default=datetime.utcnow)
    productID = db.Column(db.Integer, db.ForeignKey('productMeta.id'), nullable=False)
    warehouseID = db.Column(db.Integer, nullable=False)
    quantity = db.Column(db.Integer, nullable=False)
    category = db.Column(db.String(50), nullable=False)

class RoleMapping(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    UID = db.Column(db.String(50), nullable=False)
    role = db.Column(db.String(50), nullable=False)

# Decorator for role-based access control
def role_required(*roles):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Replace this with your actual user authentication mechanism
            user_uid = 'user123'
            user_role = RoleMapping.query.filter_by(UID=user_uid).first().role
            if user_role not in roles:
                return jsonify({'error': 'Unauthorized access'}), 403
            return func(*args, **kwargs)
        return wrapper
    return decorator

# API to get available products from a seller in various weeks
@app.route('/get_products', methods=['GET'])
@role_required('PM', 'RM')
def get_products():
    start_date = datetime.strptime(request.args.get('start_date'), '%Y-%m-%d')
    end_date = datetime.strptime(request.args.get('end_date'), '%Y-%m-%d')
    seller_name = request.args.get('seller_name')

    result = (
        db.session.query(
            func.strftime('%U', InventoryLog.addInventoryDate).label('week'),
            func.sum(InventoryLog.quantity).label('total_quantity')
        )
        .join(ProductMeta)
        .join(SellerMeta)
        .filter(SellerMeta.sellerName == seller_name)
        .filter(InventoryLog.addInventoryDate.between(start_date, end_date))
        .group_by('week')
        .all()
    )

    response = {f'W{row.week}': row.total_quantity for row in result}

    return jsonify(response)

# API to register a new seller and its first product
@app.route('/register_seller', methods=['POST', 'PUT'])
@role_required('PM', 'RM')
def register_seller():
    data = request.json

    new_seller = SellerMeta(
        sellerName=data['seller_name'],
        sellerGST=data['seller_gst'],
        onboardingDate=datetime.utcnow()
    )
    db.session.add(new_seller)
    db.session.commit()

    new_product = ProductMeta(
        productName=data['product_name'],
        onboardInventoryDate=datetime.utcnow(),
        sellerID=new_seller.id
    )
    db.session.add(new_product)

    initial_inventory = InventoryLog(
        productID=new_product.id,
        warehouseID=data['warehouse_id'],
        quantity=100,
        category=data['category']
    )
    db.session.add(initial_inventory)

    db.session.commit()

    return jsonify({'message': 'Seller and product registered successfully'})

if __name__ == '__main__':
    db.create_all()
    app.run(debug=True)
