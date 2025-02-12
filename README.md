# Retail Sales Data Pipeline

import pandas as pd
import numpy as np
import pymysql
from flask import Flask, jsonify
import boto3
import urllib3
import logging

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Fix SSL issue by ensuring urllib3 uses a secure SSL context
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# AWS S3 Configuration
S3_BUCKET = 'de-sales-data-1'
S3_FILE = 'sales_data.csv'
s3_client = boto3.client('s3')

def extract_data():
    """Extracts sales data from S3 with error handling"""
    try:
        logging.info("Extracting data from S3")
        s3_client.download_file(S3_BUCKET, S3_FILE, 'sales_data.csv')
        df = pd.read_csv('sales_data.csv')
        logging.info("Data extraction successful")
        return df
    except Exception as e:
        logging.error(f"Error extracting data: {e}")
        return pd.DataFrame()

def transform_data(df):
    """Cleans and transforms the data"""
    if df.empty:
        logging.warning("No data available for transformation.")
        return df
    logging.info("Transforming data")
    df.dropna(inplace=True)
    df['Quantity'] = pd.to_numeric(df['Quantity'], errors='coerce').fillna(0).astype(int)
    df['UnitPrice'] = pd.to_numeric(df['UnitPrice'], errors='coerce').fillna(0.0)
    df['TotalPrice'] = df['Quantity'] * df['UnitPrice']
    logging.info("Data transformation complete")
    return df

def load_data(df):
    """Loads transformed data into MySQL database with error handling"""
    if df.empty:
        logging.warning("No data available for loading.")
        return
    try:
        logging.info("Connecting to MySQL database")
        connection = pymysql.connect(
            host='localhost',
            user='root',
            password='1234567',
            database='retail_db'
        )
        cursor = connection.cursor()
        for _, row in df.iterrows():
            cursor.execute(
                """
                INSERT INTO sales (OrderID, Product, Quantity, UnitPrice, TotalPrice)
                VALUES (%s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                    Product = VALUES(Product),
                    Quantity = VALUES(Quantity),
                    UnitPrice = VALUES(UnitPrice),
                    TotalPrice = VALUES(TotalPrice)
                """,
                (row['OrderID'], row['Product'], row['Quantity'], row['UnitPrice'], row['TotalPrice'])
            )
        connection.commit()
        cursor.close()
        connection.close()
        logging.info("Data successfully loaded into MySQL")
    except Exception as e:
        logging.error(f"Error loading data: {e}")
        connection.rollback()
        cursor.close()
        connection.close()

# Flask API
app = Flask(__name__)

@app.route("/")
def home():
    return "Retail Sales Data Pipeline is Running!"

def get_sales():
    try:
        logging.info("Fetching sales data from MySQL")
        connection = pymysql.connect(
            host='your-mysql-host',
            user='your-user',
            password='your-password',
            database='retail_db'
        )
        cursor = connection.cursor(pymysql.cursors.DictCursor)
        cursor.execute("SELECT * FROM sales")
        data = cursor.fetchall()
        cursor.close()
        connection.close()
        logging.info("Sales data fetched successfully")
        return jsonify(data)
    except Exception as e:
        logging.error(f"Error fetching sales data: {e}")
        return jsonify({"error": str(e)})

@app.route('/health', methods=['GET'])
def health_check():
    return jsonify({"status": "healthy"})

if __name__ == '__main__':
    df = extract_data()
    df = transform_data(df)
    load_data(df)
    app.run(debug=True)
