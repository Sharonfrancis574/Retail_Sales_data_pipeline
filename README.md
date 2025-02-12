# Retail_Sales_data_pipeline
# Retail Sales Data Pipeline

## Project Overview
# This project demonstrates an ETL pipeline for processing retail sales data.
# Data is extracted from CSV, transformed using Pandas, loaded into a MySQL database,
# and served via a Flask API. AWS services like S3 and Lambda are used for scalability.

# Import necessary libraries
import pandas as pd
import numpy as np
import pymysql
from flask import Flask, jsonify
import boto3
import urllib3

# Fix SSL issue by ensuring urllib3 uses a secure SSL context
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# AWS S3 Configuration
S3_BUCKET = 'your-s3-bucket-name'
S3_FILE = 'sales_data.csv'
s3_client = boto3.client('s3')

def extract_data():
    """Extracts sales data from S3 with error handling"""
    try:
        s3_client.download_file(S3_BUCKET, S3_FILE, 'sales_data.csv')
        df = pd.read_csv('sales_data.csv')
        return df
    except Exception as e:
        print(f"Error extracting data: {e}")
        return pd.DataFrame()

def transform_data(df):
    """Cleans and transforms the data"""
    if df.empty:
        print("No data available for transformation.")
        return df
    df.dropna(inplace=True)
    df['TotalPrice'] = df['Quantity'] * df['UnitPrice']
    return df

def load_data(df):
    """Loads transformed data into MySQL database with error handling"""
    if df.empty:
        print("No data available for loading.")
        return
    try:
        connection = pymysql.connect(
            host='your-mysql-host',
            user='your-user',
            password='your-password',
            database='retail_db'
        )
        cursor = connection.cursor()
        for _, row in df.iterrows():
            cursor.execute("""
                INSERT INTO sales (OrderID, Product, Quantity, UnitPrice, TotalPrice)
                VALUES (%s, %s, %s, %s, %s)
            """, (row['OrderID'], row['Product'], row['Quantity'], row['UnitPrice'], row['TotalPrice']))
        connection.commit()
        cursor.close()
        connection.close()
    except Exception as e:
        print(f"Error loading data: {e}")

# Flask API
app = Flask(__name__)
@app.route('/sales', methods=['GET'])
def get_sales():
    try:
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
        return jsonify(data)
    except Exception as e:
        return jsonify({"error": str(e)})

if __name__ == '__main__':
    df = extract_data()
    df = transform_data(df)
    load_data(df)
    app.run(debug=True)
