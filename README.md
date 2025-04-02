# Building a Text-to-SQL Application in Python
## A Guide for Java Developers

This guide will help you build a basic Text-to-SQL application in Python, leveraging your existing knowledge of Java while introducing Python-specific concepts.

## Prerequisites

- Basic Python installation (Python 3.8+ recommended)
- Familiarity with installing Python packages
- Access to a database (SQLite will be used in this example for simplicity)

## Project Setup

### 1. Set Up a Virtual Environment

Python's virtual environments are similar to Java's Maven/Gradle dependency management:

```bash
# Create a virtual environment
python -m venv text2sql-env

# Activate it (Windows)
text2sql-env\Scripts\activate

# Activate it (Mac/Linux)
source text2sql-env/bin/activate
```

### 2. Install Required Libraries

```bash
pip install flask sqlalchemy nltk spacy transformers pandas sqlite3
```

### 3. Project Structure

```
text2sql-project/
├── app.py              # Main application file
├── database.py         # Database connection and schema
├── text2sql/
│   ├── __init__.py    
│   ├── parser.py       # NL parsing logic
│   ├── sql_generator.py # SQL generation
│   └── utils.py        # Utility functions
├── templates/          # HTML templates for web interface
│   └── index.html
├── static/             # Static assets
└── requirements.txt    # Dependencies file
```

## Step 1: Define Your Database

Create a file `database.py`:

```python
from sqlalchemy import create_engine, Column, Integer, String, Float, MetaData, Table
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# Create an engine that connects to the SQLite database
engine = create_engine('sqlite:///sales.db', echo=True)
Base = declarative_base()

# Define a sample table (equivalent to a Java class with @Entity)
class Product(Base):
    __tablename__ = 'products'
    
    id = Column(Integer, primary_key=True)
    name = Column(String)
    category = Column(String)
    price = Column(Float)
    inventory = Column(Integer)
    
    def __repr__(self):
        return f"<Product(name='{self.name}', price={self.price})>"

# Create the tables
Base.metadata.create_all(engine)

# Create a session factory (similar to EntityManager in JPA)
Session = sessionmaker(bind=engine)

# Function to get metadata about the database schema
def get_schema_metadata():
    """Return metadata about the database schema for NL processing"""
    metadata = {}
    for table in Base.metadata.tables.values():
        columns = []
        for column in table.columns:
            columns.append({
                'name': column.name,
                'type': str(column.type),
                'primary_key': column.primary_key
            })
        metadata[table.name] = {
            'columns': columns
        }
    return metadata

# Add some sample data
def add_sample_data():
    session = Session()
    
    # Check if data already exists
    if session.query(Product).count() == 0:
        products = [
            Product(name="Laptop", category="Electronics", price=999.99, inventory=45),
            Product(name="Smartphone", category="Electronics", price=699.99, inventory=120),
            Product(name="Coffee Maker", category="Appliances", price=89.99, inventory=30),
            Product(name="Desk Chair", category="Furniture", price=199.99, inventory=15),
            Product(name="Headphones", category="Electronics", price=149.99, inventory=75)
        ]
        session.add_all(products)
        session.commit()
    
    session.close()
```

## Step 2: Create the Natural Language to SQL Parser

Create `text2sql/parser.py`:

```python
import spacy
import re
from .utils import preprocess_text

# Load the English language model
nlp = spacy.load("en_core_web_sm")

class NLParser:
    def __init__(self, schema_metadata):
        """
        Initialize the parser with database schema metadata
        """
        self.schema = schema_metadata
        self.tables = list(schema_metadata.keys())
        
    def extract_intent(self, query):
        """
        Extract the primary intent from the query
        Similar to identifying CRUD operations
        """
        query = query.lower()
        
        # Identify the operation type
        if any(word in query for word in ["show", "display", "get", "find", "list", "what", "which"]):
            return "SELECT"
        elif any(word in query for word in ["add", "create", "insert"]):
            return "INSERT"
        elif any(word in query for word in ["change", "update", "modify", "set"]):
            return "UPDATE"
        elif any(word in query for word in ["delete", "remove", "drop"]):
            return "DELETE"
        else:
            return "SELECT"  # Default to SELECT
    
    def identify_table(self, query):
        """
        Identify which table(s) the query is referring to
        """
        query = query.lower()
        identified_tables = []
        
        # Check for plural or singular forms of table names
        for table in self.tables:
            singular = table.rstrip('s')  # Handle potential plural form
            if table in query or singular in query:
                identified_tables.append(table)
        
        # If no tables found, assume the first table
        if not identified_tables and self.tables:
            identified_tables.append(self.tables[0])
            
        return identified_tables
    
    def extract_conditions(self, query):
        """
        Extract conditions from the query
        Similar to WHERE clause conditions
        """
        # Simple condition extraction
        conditions = []
        doc = nlp(query)
        
        # Look for patterns like: where price is less than 100
        comparison_terms = {
            "greater than": ">",
            "more than": ">",
            "higher than": ">",
            "less than": "<",
            "lower than": "<",
            "equal to": "=",
            "equals": "=",
            "is": "="
        }
        
        for term, operator in comparison_terms.items():
            if term in query.lower():
                # Find the value after the comparison term
                pattern = f"{term}\\s+(\\d+)"
                matches = re.search(pattern, query.lower())
                if matches:
                    value = matches.group(1)
                    
                    # Try to find what column this applies to
                    potential_columns = []
                    for table in self.schema:
                        for column in self.schema[table]['columns']:
                            if column['name'] in query.lower():
                                conditions.append({
                                    'column': column['name'], 
                                    'operator': operator, 
                                    'value': value
                                })
        
        return conditions
    
    def extract_columns(self, query, table):
        """
        Determine which columns to select
        """
        query = query.lower()
        columns = []
        
        if "all" in query or "*" in query:
            return ["*"]
        
        # Check for column names in the query
        if table in self.schema:
            for column in self.schema[table]['columns']:
                if column['name'] in query:
                    columns.append(column['name'])
        
        # Default to all columns if none specified
        if not columns:
            return ["*"]
            
        return columns
    
    def parse(self, query):
        """
        Parse the natural language query into structured components
        """
        # Preprocess the query
        processed_query = preprocess_text(query)
        
        # Extract basic components
        intent = self.extract_intent(processed_query)
        tables = self.identify_table(processed_query)
        
        result = {
            'intent': intent,
            'tables': tables,
            'parsed': True
        }
        
        # Add operation-specific components
        if intent == "SELECT":
            if tables:
                result['columns'] = self.extract_columns(processed_query, tables[0])
                result['conditions'] = self.extract_conditions(processed_query)
        
        return result
```

## Step 3: Create the SQL Generator

Create `text2sql/sql_generator.py`:

```python
class SQLGenerator:
    def __init__(self):
        pass
        
    def generate_sql(self, parsed_query):
        """
        Generate SQL from the parsed query components
        """
        if not parsed_query.get('parsed', False):
            return "-- Could not parse the query"
            
        intent = parsed_query.get('intent')
        
        if intent == "SELECT":
            return self._generate_select(parsed_query)
        elif intent == "INSERT":
            return self._generate_insert(parsed_query)
        elif intent == "UPDATE":
            return self._generate_update(parsed_query)
        elif intent == "DELETE":
            return self._generate_delete(parsed_query)
        else:
            return "-- Unsupported query type"
            
    def _generate_select(self, parsed_query):
        """Generate a SELECT statement"""
        tables = parsed_query.get('tables', [])
        columns = parsed_query.get('columns', ['*'])
        conditions = parsed_query.get('conditions', [])
        
        if not tables:
            return "-- No tables identified"
            
        # Build the base query
        columns_str = ", ".join(columns)
        sql = f"SELECT {columns_str} FROM {tables[0]}"
        
        # Add WHERE clause if there are conditions
        if conditions:
            where_clauses = []
            for condition in conditions:
                col = condition.get('column')
                op = condition.get('operator')
                val = condition.get('value')
                
                if all([col, op, val]):
                    where_clauses.append(f"{col} {op} {val}")
            
            if where_clauses:
                sql += " WHERE " + " AND ".join(where_clauses)
                
        return sql + ";"
        
    def _generate_insert(self, parsed_query):
        # For simplicity, just returning a template
        tables = parsed_query.get('tables', [])
        if not tables:
            return "-- No tables identified"
        
        return f"INSERT INTO {tables[0]} (...) VALUES (...);"
    
    def _generate_update(self, parsed_query):
        # For simplicity, just returning a template
        tables = parsed_query.get('tables', [])
        if not tables:
            return "-- No tables identified"
        
        return f"UPDATE {tables[0]} SET ... WHERE ...;"
    
    def _generate_delete(self, parsed_query):
        # For simplicity, just returning a template
        tables = parsed_query.get('tables', [])
        if not tables:
            return "-- No tables identified"
        
        return f"DELETE FROM {tables[0]} WHERE ...;"
```

## Step 4: Create Utilities

Create `text2sql/utils.py`:

```python
import re

def preprocess_text(text):
    """
    Clean and prepare text for processing
    """
    # Convert to lowercase
    text = text.lower()
    
    # Remove special characters except spaces and alphanumerics
    text = re.sub(r'[^\w\s]', ' ', text)
    
    # Replace multiple spaces with a single space
    text = re.sub(r'\s+', ' ', text).strip()
    
    return text

def execute_sql(sql, engine):
    """
    Execute the SQL and return results
    """
    try:
        import pandas as pd
        
        # Check if it's a SELECT query
        if sql.strip().upper().startswith("SELECT"):
            # Use pandas to execute and format results
            result = pd.read_sql_query(sql, engine)
            return {"success": True, "data": result.to_dict(orient="records")}
        else:
            # For non-SELECT queries, execute directly
            from sqlalchemy import text
            with engine.connect() as connection:
                connection.execute(text(sql))
                connection.commit()
            return {"success": True, "message": "Query executed successfully"}
    except Exception as e:
        return {"success": False, "error": str(e)}
```

## Step 5: Create the main application

Create `app.py`:

```python
from flask import Flask, request, jsonify, render_template
from text2sql.parser import NLParser
from text2sql.sql_generator import SQLGenerator
from text2sql.utils import execute_sql
from database import get_schema_metadata, engine, add_sample_data

app = Flask(__name__)

# Initialize the Text-to-SQL components
schema_metadata = get_schema_metadata()
nl_parser = NLParser(schema_metadata)
sql_generator = SQLGenerator()

# Add sample data
add_sample_data()

@app.route('/')
def index():
    """Render the main page"""
    return render_template('index.html')

@app.route('/api/query', methods=['POST'])
def process_query():
    """Process a natural language query and return SQL + results"""
    data = request.json
    nl_query = data.get('query', '')
    
    if not nl_query:
        return jsonify({"error": "No query provided"}), 400
    
    # Process the query through our pipeline
    parsed_query = nl_parser.parse(nl_query)
    sql = sql_generator.generate_sql(parsed_query)
    results = execute_sql(sql, engine)
    
    return jsonify({
        "natural_language_query": nl_query,
        "parsed_query": parsed_query,
        "sql": sql,
        "results": results
    })

if __name__ == '__main__':
    app.run(debug=True)
```

## Step 6: Create a Simple Web Interface

Create `templates/index.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Text-to-SQL Converter</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
        }
        .container {
            display: flex;
            flex-direction: column;
            gap: 20px;
        }
        .input-area {
            display: flex;
            gap: 10px;
        }
        input {
            flex-grow: 1;
            padding: 10px;
            font-size: 16px;
            border: 1px solid #ccc;
            border-radius: 4px;
        }
        button {
            padding: 10px 20px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        .results {
            border: 1px solid #ddd;
            padding: 20px;
            border-radius: 4px;
        }
        .sql-area {
            background-color: #f5f5f5;
            padding: 10px;
            border-radius: 4px;
            font-family: monospace;
        }
        table {
            width: 100%;
            border-collapse: collapse;
        }
        th, td {
            padding: 8px;
            text-align: left;
            border-bottom: 1px solid #ddd;
        }
        th {
            background-color: #f2f2f2;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Text-to-SQL Converter</h1>
        
        <div class="input-area">
            <input type="text" id="nlQuery" placeholder="Enter your question in natural language. e.g., 'Show me all products in Electronics category'" />
            <button onclick="processQuery()">Convert</button>
        </div>
        
        <div class="results" id="results">
            <p>Results will appear here...</p>
        </div>
    </div>
    
    <script>
        async function processQuery() {
            const nlQuery = document.getElementById('nlQuery').value;
            const resultsDiv = document.getElementById('results');
            
            if (!nlQuery) {
                alert('Please enter a query');
                return;
            }
            
            resultsDiv.innerHTML = '<p>Processing...</p>';
            
            try {
                const response = await fetch('/api/query', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({ query: nlQuery }),
                });
                
                const data = await response.json();
                
                if (data.error) {
                    resultsDiv.innerHTML = `<p>Error: ${data.error}</p>`;
                    return;
                }
                
                let html = `
                    <h3>Generated SQL:</h3>
                    <div class="sql-area">${data.sql}</div>
                `;
                
                if (data.results.success) {
                    if (data.results.data) {
                        // Results table
                        html += `<h3>Results:</h3>`;
                        
                        if (data.results.data.length > 0) {
                            const columns = Object.keys(data.results.data[0]);
                            
                            html += `
                                <table>
                                    <thead>
                                        <tr>
                                            ${columns.map(col => `<th>${col}</th>`).join('')}
                                        </tr>
                                    </thead>
                                    <tbody>
                            `;
                            
                            data.results.data.forEach(row => {
                                html += `
                                    <tr>
                                        ${columns.map(col => `<td>${row[col]}</td>`).join('')}
                                    </tr>
                                `;
                            });
                            
                            html += `
                                    </tbody>
                                </table>
                            `;
                        } else {
                            html += `<p>No results found.</p>`;
                        }
                    } else if (data.results.message) {
                        html += `<p>${data.results.message}</p>`;
                    }
                } else {
                    html += `<p>Error executing SQL: ${data.results.error}</p>`;
                }
                
                resultsDiv.innerHTML = html;
            } catch (error) {
                resultsDiv.innerHTML = `<p>Error: ${error.message}</p>`;
            }
        }
    </script>
</body>
</html>
```

## Running Your Application

1. Create a `requirements.txt` file with all dependencies:
```
flask==2.0.1
sqlalchemy==1.4.23
nltk==3.6.3
spacy==3.1.3
transformers==4.11.3
pandas==1.3.3
```

2. Install spaCy's English language model:
```bash
python -m spacy download en_core_web_sm
```

3. Run the application:
```bash
python app.py
```

4. Open your browser to `http://localhost:5000`

## Example Queries to Try

- "Show me all products"
- "Find all electronics products"
- "What products cost less than 200"
- "Show me products with inventory greater than 50"

## From Here: Enhancing Your Application

1. **Improve the NL parser**: Integrate more advanced NLP techniques or a pre-trained model
2. **Add more database operations**: Support for aggregations, joins, and more complex queries
3. **Error handling**: Add more robust error handling and user feedback
4. **User interface improvements**: Add history, suggested queries, etc.
5. **Documentation**: Add API documentation using Swagger or a similar tool

## Java vs Python: Key Differences to Note

1. **Dynamic typing**: Python is dynamically typed unlike Java
2. **Indentation**: Python uses indentation for code blocks, not curly braces
3. **No explicit interfaces**: Python uses "duck typing" instead of interfaces
4. **Package management**: pip vs Maven/Gradle
5. **Web frameworks**: Flask/FastAPI vs Spring Boot
