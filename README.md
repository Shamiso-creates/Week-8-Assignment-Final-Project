-- Create the database
CREATE DATABASE IF NOT EXISTS EcommerceStore;
USE EcommerceStore;

-- Customers table
CREATE TABLE Customers (
    customer_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    date_created TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP NULL,
    CONSTRAINT chk_email CHECK (email LIKE '%@%.%')
);

-- Addresses table (One-to-Many with Customers)
CREATE TABLE Addresses (
    address_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    address_type ENUM('billing', 'shipping') NOT NULL,
    street_address VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100),
    postal_code VARCHAR(20) NOT NULL,
    country VARCHAR(100) NOT NULL,
    is_default BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (customer_id) REFERENCES Customers(customer_id) ON DELETE CASCADE,
    INDEX idx_customer_address (customer_id, address_type)
);

-- Categories table
CREATE TABLE Categories (
    category_id INT AUTO_INCREMENT PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL UNIQUE,
    parent_category_id INT NULL,
    description TEXT,
    FOREIGN KEY (parent_category_id) REFERENCES Categories(category_id) ON DELETE SET NULL,
    INDEX idx_parent_category (parent_category_id)
);

-- Products table
CREATE TABLE Products (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(255) NOT NULL,
    description TEXT,
    category_id INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
    cost DECIMAL(10, 2) CHECK (cost >= 0),
    sku VARCHAR(100) UNIQUE NOT NULL,
    weight DECIMAL(8, 2) CHECK (weight >= 0),
    stock_quantity INT NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    reorder_level INT DEFAULT 5 CHECK (reorder_level >= 0),
    is_active BOOLEAN DEFAULT TRUE,
    date_added TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES Categories(category_id) ON DELETE RESTRICT,
    INDEX idx_category (category_id),
    INDEX idx_sku (sku)
);

-- Product attributes table (for variable product types)
CREATE TABLE ProductAttributes (
    product_id INT NOT NULL,
    attribute_name VARCHAR(100) NOT NULL,
    attribute_value TEXT NOT NULL,
    PRIMARY KEY (product_id, attribute_name),
    FOREIGN KEY (product_id) REFERENCES Products(product_id) ON DELETE CASCADE
);

-- Product images table
CREATE TABLE ProductImages (
    image_id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    image_url VARCHAR(255) NOT NULL,
    alt_text VARCHAR(255),
    is_primary BOOLEAN DEFAULT FALSE,
    display_order INT DEFAULT 0,
    FOREIGN KEY (product_id) REFERENCES Products(product_id) ON DELETE CASCADE,
    INDEX idx_product_images (product_id)
);

-- Orders table
CREATE TABLE Orders (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    order_status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded') DEFAULT 'pending',
    total_amount DECIMAL(10, 2) NOT NULL CHECK (total_amount >= 0),
    tax_amount DECIMAL(10, 2) DEFAULT 0 CHECK (tax_amount >= 0),
    shipping_amount DECIMAL(10, 2) DEFAULT 0 CHECK (shipping_amount >= 0),
    shipping_address_id INT NOT NULL,
    billing_address_id INT NOT NULL,
    notes TEXT,
    FOREIGN KEY (customer_id) REFERENCES Customers(customer_id) ON DELETE RESTRICT,
    FOREIGN KEY (shipping_address_id) REFERENCES Addresses(address_id),
    FOREIGN KEY (billing_address_id) REFERENCES Addresses(address_id),
    INDEX idx_customer_orders (customer_id),
    INDEX idx_order_status (order_status)
);

-- Order items table (Many-to-Many between Orders and Products)
CREATE TABLE OrderItems (
    order_item_id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10, 2) NOT NULL CHECK (unit_price >= 0),
    subtotal DECIMAL(10, 2) GENERATED ALWAYS AS (quantity * unit_price) STORED,
    FOREIGN KEY (order_id) REFERENCES Orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES Products(product_id) ON DELETE RESTRICT,
    INDEX idx_order_items (order_id)
);

-- Payments table
CREATE TABLE Payments (
    payment_id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    payment_method ENUM('credit_card', 'debit_card', 'paypal', 'bank_transfer') NOT NULL,
    payment_status ENUM('pending', 'completed', 'failed', 'refunded') DEFAULT 'pending',
    amount DECIMAL(10, 2) NOT NULL CHECK (amount >= 0),
    transaction_id VARCHAR(255) UNIQUE,
    payment_date TIMESTAMP NULL,
    FOREIGN KEY (order_id) REFERENCES Orders(order_id) ON DELETE CASCADE,
    INDEX idx_order_payments (order_id)
);

-- Reviews table
CREATE TABLE Reviews (
    review_id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    customer_id INT NOT NULL,
    rating TINYINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title VARCHAR(255),
    comment TEXT,
    review_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_approved BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (product_id) REFERENCES Products(product_id) ON DELETE CASCADE,
    FOREIGN KEY (customer_id) REFERENCES Customers(customer_id) ON DELETE CASCADE,
    UNIQUE KEY unique_product_customer_review (product_id, customer_id),
    INDEX idx_product_reviews (product_id)
);

-- Shopping cart table
CREATE TABLE ShoppingCart (
    cart_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL UNIQUE,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES Customers(customer_id) ON DELETE CASCADE
);

-- Cart items table
CREATE TABLE CartItems (
    cart_item_id INT AUTO_INCREMENT PRIMARY KEY,
    cart_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    added_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (cart_id) REFERENCES ShoppingCart(cart_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES Products(product_id) ON DELETE CASCADE,
    UNIQUE KEY unique_cart_product (cart_id, product_id),
    INDEX idx_cart_items (cart_id)
);

-- Coupons table
CREATE TABLE Coupons (
    coupon_id INT AUTO_INCREMENT PRIMARY KEY,
    coupon_code VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    discount_type ENUM('percentage', 'fixed') NOT NULL,
    discount_value DECIMAL(10, 2) NOT NULL CHECK (discount_value >= 0),
    minimum_order DECIMAL(10, 2) DEFAULT 0 CHECK (minimum_order >= 0),
    max_uses INT,
    uses_count INT DEFAULT 0 CHECK (uses_count >= 0),
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    CHECK (end_date >= start_date)
);

-- Order coupons table (Many-to-Many between Orders and Coupons)
CREATE TABLE OrderCoupons (
    order_id INT NOT NULL,
    coupon_id INT NOT NULL,
    discount_amount DECIMAL(10, 2) NOT NULL CHECK (discount_amount >= 0),
    PRIMARY KEY (order_id, coupon_id),
    FOREIGN KEY (order_id) REFERENCES Orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (coupon_id) REFERENCES Coupons(couple_id) ON DELETE RESTRICT
);

-- Inventory log table
CREATE TABLE InventoryLog (
    log_id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    change_type ENUM('in', 'out', 'adjustment', 'return') NOT NULL,
    quantity_change INT NOT NULL,
    new_stock_level INT NOT NULL,
    reason VARCHAR(255),
    log_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES Products(product_id) ON DELETE CASCADE,
    INDEX idx_product_inventory (product_id)
);

-- Create a view for product details
CREATE VIEW ProductDetails AS
SELECT 
    p.product_id,
    p.product_name,
    p.description,
    c.category_name,
    p.price,
    p.cost,
    p.sku,
    p.weight,
    p.stock_quantity,
    p.reorder_level,
    p.is_active,
    (SELECT image_url FROM ProductImages WHERE product_id = p.product_id AND is_primary = TRUE LIMIT 1) AS primary_image,
    AVG(r.rating) AS average_rating,
    COUNT(r.review_id) AS review_count
FROM Products p
JOIN Categories c ON p.category_id = c.category_id
LEFT JOIN Reviews r ON p.product_id = r.product_id AND r.is_approved = TRUE
GROUP BY p.product_id;

-- Create a view for order summaries
CREATE VIEW OrderSummaries AS
SELECT 
    o.order_id,
    o.order_date,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    o.order_status,
    o.total_amount,
    o.tax_amount,
    o.shipping_amount,
    p.payment_status 
    COUNT(oi.order_item_id) AS item_count
FROM Orders o
JOIN Customers c ON o.customer_id = c.customer_id
LEFT JOIN Payments p ON o.order_id = p.order_id
LEFT JOIN OrderItems oi ON o.order_id = oi.order_id
GROUP BY o.order_id;

-- Create indexes for better performance
CREATE INDEX idx_products_category ON Products(category_id);
CREATE INDEX idx_orders_customer_date ON Orders(customer_id, order_date);
CREATE INDEX idx_order_items_product ON OrderItems(product_id);
CREATE INDEX idx_payments_status ON Payments(payment_status);
CREATE INDEX idx_reviews_product_rating ON Reviews(product_id, rating);
CREATE INDEX idx_customers_email ON Customers(email);
CREATE INDEX idx_products_price_stock ON Products(price, stock_quantity); 
