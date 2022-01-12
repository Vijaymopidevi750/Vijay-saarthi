CREATE DATABASE IF NOT EXISTS books_data;
USE books_data;
CREATE TABLE book (
    id int NOT NULL AUTO_INCREMENT,
    name varchar(255) NOT NULL,
    isbn varchar(50) NOT NULL,
    authors varchar(255) NOT NULL,
    country varchar(80) NOT NULL,
    number_of_pages int NOT NULL,
    publisher varchar(255) NOT NULL,
    release_date datetime,
    PRIMARY KEY (id)
);

 111  book_api.py 
@@ -0,0 +1,111 @@
import flask
import requests
import json

from mysqlconnect import *

app = flask.Flask(__name__)

@app.route("/")
def test():
    return "Hello"

def output_data(status_code, status, message=None):
    result = {
               "status_code": status_code,
               "status": status,
               "data": []
             }
    if message:
        result['message'] = message
    return result

@app.route("/api/external-books")
def get_book_external():
    url = "https://www.anapioficeandfire.com/api/books"
    book_name =flask.request.args.get('name')
    if book_name:
        url = "https://www.anapioficeandfire.com/api/books?name={}".format(book_name)
        response = requests.get(url)
        if response.status_code == 200:
            data = json.loads(response.content.decode('utf-8'))
            if data:
                result = output_data(response.status_code, "success")
                result['data'].append({
                         "name": data[0].get("name"),
                         "isbn": data[0].get("isbn"),
                         "authors": data[0].get("authors"),
                         "numberOfPages": data[0].get("numberOfPages"),
                         "publisher": data[0].get("publisher"),
                         "country": data[0].get("country"),
                         "release_date": data[0].get("released")
                        })
                return flask.jsonify(result)
            else:
                return flask.jsonify(output_data(response.status_code, "success"))
        else:
            return "External Api returned error :{}".format(response.status_code)
    else:
        return "Please provide a valid Book name"

@app.route("/api/v1/books", methods=['GET', 'POST'])
def books():
    db = dbConnection()
    if flask.request.method == 'POST':
        data = flask.request.get_json()
        status = db.insert(data)
        db.db_close()
        if status == "success":
            result = output_data(201, status)
            result['data'].append(data)
            return flask.jsonify(result), 201
        else:
            return flask.jsonify(output_data(200, "fail"))
    if flask.request.method == 'GET':
        data = db.select()
        result = output_data(200, "success")
        result['data'] = data
        db.db_close()
        return flask.jsonify(result), 200

@app.route("/api/v1/books/<int:book_id>", methods=['GET', 'PATCH', "DELETE"])
def update_book(book_id):
    db = dbConnection()
    if flask.request.method == 'GET':
        data = db.select(book_id)
        result  = output_data(200, "success")
        result['data'] = data
        db.db_close()
        return flask.jsonify(result), 200
    if flask.request.method == 'PATCH':
        data = flask.request.get_json()
        status = db.update(data, book_id)
        if status == "success":
            data = db.select(book_id)
            message = "The book {} was updated successfully".format(data.get('name'))
            result = output_data(200, "success", message)
            result['data'] = data
            db.db_close()
            return flask.jsonify(result), 200
        else:
            db.db.close()
            return flask.jsonify(output_data(200, "fail")) 

    if flask.request.method == 'DELETE':
        data = db.select(book_id)
        if data:
            status = db.delete(book_id)
            if status == "success":
                message = "The book {} was deleted succssfully".format(data.get("name"))
                db.db_close()
                return flask.jsonify(output_data(200, "success", message))
            else:
                message = status
                db.db_close()
                return flask.jsonify(output_data(200, "fail", message))
        else:
            message = "Provide a valid book id"
            return flask.jsonify(output_data(200, "fail", message))

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
 69  mysqlconnect.py 
@@ -0,0 +1,69 @@
import pymysql
import collections

class dbConnection:
    def __init__(self):
        try:
            self.conn = pymysql.connect(host="localhost", user="root", password="hello", db="books_data")
        except Exception as e:
            print(e)

    def insert(self, data):
        sql = """INSERT INTO book(name, isbn, authors, country, number_of_pages, publisher, release_date) VALUES 
                 ('{}', '{}', '{}', '{}', '{}', '{}', '{}')""".format(data.get("name"), data.get("isbn"), 
                 ",".join(data.get("authors")), data.get("country"), data.get("number_of_pages"), data.get("publisher"), 
                 data.get("release_date"))
        print(sql)
        try:
            cur = self.conn.cursor()
            cur.execute(sql)
            self.conn.commit()
            return "success"
        except Exception as e:
            print(e)
            return e

    def select(self, book_id=None):
        sql = "SELECT * FROM book"
        if book_id:
            sql = "SELECT * FROM book WHERE id = {}".format(book_id)
        try:
            cur = self.conn.cursor()
            cur.execute(sql)
            headers = list(map(lambda x: x[0], cur.description))
            data = []
            for value in cur.fetchall():
                book_dict = collections.OrderedDict(zip(headers, value))
                book_dict['authors'] = book_dict['authors'].split(",")
                data.append(book_dict)
            if book_id and data:
                return data[0]
            return data
        except Exception as e:
            print(e)

    def update(self, data, book_id):
        temp = ", ".join(["{} = '{}'".format(key, data[key]) for key in data ])
        sql = """UPDATE book SET {} WHERE id = {}""".format(temp, book_id)
        try:
            cur = self.conn.cursor()
            cur.execute(sql)
            self.conn.commit()
            return "success"
        except Exception as e:
            print(e)
            return e

    def delete(self, book_id):
        sql = "DELETE FROM book WHERE id = {}".format(book_id)
        try:
            cur = self.conn.cursor()
            cur.execute(sql)
            self.conn.commit()
            return "success"
        except Exception as e:
            print(e)
            return e        

    def db_close(self):
        self.conn.close()
