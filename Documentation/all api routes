import pandas as pd
from flask.cli import load_dotenv
from flask_cors import CORS  # Import the CORS extension

from functools import wraps
from flask import Flask, jsonify, Response, request, redirect, url_for
import flask
import os

from sqlalchemy import create_engine
from vanna.flask import MemoryCache
from vanna.remote import VannaDefault

load_dotenv()
app = Flask(__name__, static_url_path='')
CORS(app)  # Enable CORS for all routes in the app

# SETUP
cache = MemoryCache()

# from vanna.local import LocalContext_OpenAI
# vn = LocalContext_OpenAI()


api_key = 'e20497935e1f41f1954c93b3a62d224d'

vanna_model_name = 'ranjits-ai'
vn = VannaDefault(model=vanna_model_name, api_key=api_key)

# Connection details
database = "canigo"
username = "root"
password = ""
host = "localhost"
port = '3306'

# Create a SQLAlchemy engine
engine = create_engine(f"mysql+pymysql://{username}:{password}@{host}:{port}/{database}")


# You define a function that takes in a SQL query as a string and returns a pandas dataframe
def run_sql(sql: str) -> pd.DataFrame:
    df = pd.read_sql_query(sql, engine)
    return df


# This gives the package a function that it can use to run the SQL
vn.run_sql = run_sql
vn.run_sql_is_set = True

df_information_schema = vn.run_sql("SELECT * FROM INFORMATION_SCHEMA.COLUMNS")


# NO NEED TO CHANGE ANYTHING BELOW THIS LINE
def requires_cache(fields):
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            id = request.args.get('id')

            if id is None:
                return jsonify({"type": "error", "error": "No id provided"})

            for field in fields:
                if cache.get(id=id, field=field) is None:
                    return jsonify({"type": "error", "error": f"No {field} found"})

            field_values = {field: cache.get(id=id, field=field) for field in fields}

            # Add the id to the field_values
            field_values['id'] = id

            return f(*args, **field_values, **kwargs)

        return decorated

    return decorator


# Function to run SQL query and return results as JSON
def run_sql_query(sql):
    try:
        # Execute SQL query and fetch results
        df = pd.read_sql_query(sql, engine)
        # Convert DataFrame to JSON
        result_json = df.to_json(orient="records")
        return result_json
    except Exception as e:
        return jsonify({"error": str(e)})


@app.route('/api/v0/generate_sql', methods=['GET'])
def generate_sql():
    question = flask.request.args.get('question')

    if question is None:
        return jsonify({"type": "error", "error": "No question provided"})

    id = cache.generate_id(question=question)
    sql_query = vn.generate_sql(question=question)

    cache.set(id=id, field='question', value=question)
    cache.set(id=id, field='sql', value=sql_query)

    result = run_sql_query(sql_query)
    return result



@app.route('/')
def root():
    return app.send_static_file('index.html')


if __name__ == '__main__':
    app.run(debug=True)
