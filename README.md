from bs4 import BeautifulSoup
import requests
import pandas as pd
import numpy as np
import sqlite3
from datetime import datetime
# Importing the required libraries

def log_progress(message):
    ''' This function logs the mentioned message of a given stage of the
    code execution to a log file. Function returns nothing'''

    timestamp_format = '%Y-%b-%d-%H:%M:%S'
    now = datetime.now()
    timestamp = now.strftime(timestamp_format)
    with open('code_log.txt', 'a') as f:
        f.write(f'{datetime.now()}: {message}\n')

def extract(url, table_attribs):
    ''' This function aims to extract the required
    information from the website and save it to a data frame. The
    function returns the data frame for further processing. '''

    page = requests.get(url).text
    data = BeautifulSoup(page, 'html.parser')
    df = pd.DataFrame(columns=table_attribs)
    tables = data.find_all('tbody')
    rows = tables[2].find_all('tr')
    for row in rows:
        col = row.find_all('td')
        if len(col)!=0:
            if col[0].find('a') is not None and '-' not in col[2]:
                data_dict = {"Name": col[0].a.contents[0], "MC_USD_Billion": col[2].contents[0]}
                df1 = pd.DataFrame(data_dict, index=[0])
                df = pd.concat([df, df1], ignore_index=True)

    USD_list = list(df['MC_USD_Billion'])
    USD_list = [float(''.join(x.split('\n'))) for x in USD_list]
    df['MC_USD_Billion'] = USD_list

    log_progress('Data extraction complete. Initiating Transfromation process')

    return df

def transform (df, csv_path):
    exchange_rate = pd.read_csv(csv_path)
    exchange_rate = exchange_rate.set_index('Currency').to_dict()['Rate']
    df['MC_USD_Billion']= [np.round(x*exchange_rate['EUR'],2) for x in df['MC_USD_Billion']]
    df['MC_USD_Billion']= [np.round(x*exchange_rate['GBP'],2) for x in df['MC_USD_Billion']]
    df['MC_USD_Billion']= [np.round(x*exchange_rate['INR'],2) for x in df['MC_USD_Billion']]
    return df

def load_to_csv(df, output_path):
    df.to_csv(csv_path)

def load_to_db(df, sql_connection, table_name):
    df.to_sql(table_name, sql_connection, if_exists = 'replace', index=False)

def run_query(query_statement, sql_connection):
    print(query_statement)
    query_output = pd.read_sql(query_statement, sql_connection)
    print(query_output)

url = 'https://web.archive.org/web/20230908091635 /https://en.wikipedia.org/wiki/List_of_largest_banks'
table_attribs = ['Name', 'MC_USD_Billion']
db_name = 'Banks.db'
table_name = 'Largest_banks'
csv_path = 'code_log.txt'

log_progress('Preliminaries complete. Initiating ETL process')

df = extract(url, table_attribs)

df = transform(df, csv_path)
log_progress('Data extraction complete. Initiating Transformation process')

load_to_csv(df, csv_path)
log_progress('Data saved to CSV file')

sql_connection = sqlite3.connect('Banks.db')
log_progress('SQL Connection initiated')

load_to_db(df, sql_connection, table_name)
log_progress('Data loaded to Database as a table. Executing queries')

query_statement = f'SELECT * FROM Largest_banks'
run_query(query_statement, sql_connection)
query_statement = f'SELECT AVG(MC_GBP_Billion) FROM Largest_banks'
run_query(query_statement, sql_connection)
query_statement = f'Select Name from Largest_banks LIMIT 5'
run_query(query_statement, sql_connection)

sql_connection.close()
log_progress('Server connection closed')
