from flask import Flask, request, jsonify
import mysql.connector
from mysql.connector import Error

app = Flask(__name__)

def create_connection():
    connection = None
    try:
        connection = mysql.connector.connect(
            host='localhost',
            user='root',
            password='',
            database='income_panel'
        )
    except Error as e:
        print(f"Error: {e}")
    return connection

@app.route('/projects', methods=['GET', 'POST'])
def manage_projects():
    connection = create_connection()
    if request.method == 'GET':
        cursor = connection.cursor(dictionary=True)
        cursor.execute("SELECT * FROM projects")
        projects_data = cursor.fetchall()
        cursor.close()
        return jsonify(projects_data)
    
    elif request.method == 'POST':
        data = request.json
        query = """INSERT INTO projects (project_name, project_status, total_income, total_expenses, profitability_percentage)
                   VALUES (%s, %s, %s, %s, %s)"""
        values = (data['project_name'], data['project_status'], data['total_income'],
                  data['total_expenses'], data['profitability_percentage'])
        cursor = connection.cursor()
        cursor.execute(query, values)
        connection.commit()
        cursor.close()
        return jsonify({"message": "Project added successfully!"}), 201

@app.route('/projects/<int:project_id>', methods=['GET', 'PUT', 'DELETE'])
def project_details(project_id):
    connection = create_connection()
    cursor = connection.cursor(dictionary=True)
    
    if request.method == 'GET':
        cursor.execute("SELECT * FROM projects WHERE id = %s", (project_id,))
        project_data = cursor.fetchone()
        cursor.close()
        return jsonify(project_data)
    
    elif request.method == 'PUT':
        data = request.json
        query = """UPDATE projects SET project_name = %s, project_status = %s, 
                   total_income = %s, total_expenses = %s, profitability_percentage = %s 
                   WHERE id = %s"""
        values = (data['project_name'], data['project_status'], data['total_income'],
                  data['total_expenses'], data['profitability_percentage'], project_id)
        cursor.execute(query, values)
        connection.commit()
        cursor.close()
        return jsonify({"message": "Project updated successfully!"})

    elif request.method == 'DELETE':
        cursor.execute("DELETE FROM projects WHERE id = %s", (project_id,))
        connection.commit()
        cursor.close()
        return jsonify({"message": "Project deleted successfully!"})

@app.route('/projects/<int:project_id>/income', methods=['POST'])
def add_income(project_id):
    connection = create_connection()
    data = request.json
    query = """INSERT INTO income (date, source, amount, category, payment_method, transaction_id, notes)
               VALUES (%s, %s, %s, %s, %s, %s, %s)"""
    values = (data['date'], data['source'], data['amount'], data['category'],
              data['payment_method'], data.get('transaction_id'), data.get('notes'))
    cursor = connection.cursor()
    cursor.execute(query, values)
    
    # Update project total income
    cursor.execute("UPDATE projects SET total_income = total_income + %s WHERE id = %s",
                   (data['amount'], project_id))
    connection.commit()
    cursor.close()
    return jsonify({"message": "Income record added successfully!"}), 201

@app.route('/projects/<int:project_id>/expenses', methods=['POST'])
def add_expense(project_id):
    connection = create_connection()
    data = request.json
    query = """INSERT INTO expenses (date, category, amount, notes, payment_method)
               VALUES (%s, %s, %s, %s, %s)"""
    values = (data['date'], data['category'], data['amount'],
              data.get('notes'), data.get('payment_method'))
    cursor = connection.cursor()
    cursor.execute(query, values)
    
    # Update project total expenses
    cursor.execute("UPDATE projects SET total_expenses = total_expenses + %s WHERE id = %s",
                   (data['amount'], project_id))
    connection.commit()
    cursor.close()
    return jsonify({"message": "Expense record added successfully!"}), 201

@app.route('/projects/<int:project_id>/profitability', methods=['GET'])
def calculate_profitability(project_id):
    connection = create_connection()
    cursor = connection.cursor(dictionary=True)
    
    query = """SELECT total_income, total_expenses, 
                      (total_income - total_expenses) / NULLIF(total_income, 0) * 100 AS profitability_percentage
               FROM projects WHERE id = %s"""
    cursor.execute(query, (project_id,))
    profitability_data = cursor.fetchone()
    cursor.close()
    
    return jsonify(profitability_data)






@app.route('/projects/filter', methods=['GET'])
def filter_projects():
    status = request.args.get('status')
    min_profitability = request.args.get('min_profitability', type=float, default=0)
    max_profitability = request.args.get('max_profitability', type=float, default=100)
    client_id = request.args.get('client_id', type=int)

    query = "SELECT * FROM projects WHERE 1=1"
    params = []

    if status:
        query += " AND project_status = %s"
        params.append(status)

    if client_id:
        query += " AND client_id = %s"
        params.append(client_id)

    query += " AND profitability_percentage BETWEEN %s AND %s"
    params.extend([min_profitability, max_profitability])

    connection = create_connection()
    cursor = connection.cursor(dictionary=True)
    cursor.execute(query, params)
    filtered_projects = cursor.fetchall()
    cursor.close()
    
    return jsonify(filtered_projects)

@app.route('/clients', methods=['GET', 'POST'])
def manage_clients():
    connection = create_connection()
    
    if request.method == 'GET':
        cursor = connection.cursor(dictionary=True)
        cursor.execute("SELECT * FROM clients")
        clients_data = cursor.fetchall()
        cursor.close()
        return jsonify(clients_data)
    
    elif request.method == 'POST':
        data = request.json
        query = "INSERT INTO clients (client_name) VALUES (%s)"
        values = (data['client_name'],)
        cursor = connection.cursor()
        cursor.execute(query, values)
        connection.commit()
        cursor.close()
        return jsonify({"message": "Client added successfully!"}), 201


if __name__ == '__main__':
    app.run(debug=True)
