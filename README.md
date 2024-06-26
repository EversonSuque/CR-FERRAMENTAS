from flask import Flask, render_template, request, redirect, url_for, flash
import sqlite3

app = Flask(__name__)
app.secret_key = 'sua_chave_secreta'


# Função para conectar ao banco de dados
def get_db_connection():
    conn = sqlite3.connect('controle_ferramentas.db')
    conn.row_factory = sqlite3.Row
    return conn


# Função para criar as tabelas se não existirem
def create_tables():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS ferramentas (
        patrimonio INTEGER PRIMARY KEY,
        nome TEXT NOT NULL,
        quantidade INTEGER NOT NULL
    )
    ''')
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS registros (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        patrimonio INTEGER,
        obra TEXT,
        FOREIGN KEY(patrimonio) REFERENCES ferramentas(patrimonio)
    )
    ''')
    conn.commit()
    conn.close()


# Chame a função para criar as tabelas
create_tables()


# Rota principal para exibir ferramentas
@app.route('/')
def index():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute('''
    SELECT f.nome, f.patrimonio, f.quantidade, r.obra
    FROM ferramentas f 
    LEFT JOIN registros r ON f.patrimonio = r.patrimonio
    ORDER BY f.nome
    ''')
    ferramentas = cursor.fetchall()
    conn.close()
    return render_template('index.html', ferramentas=ferramentas)


# Rota para adicionar ferramenta
@app.route('/adicionar', methods=('GET', 'POST'))
def adicionar():
    if request.method == 'POST':
        patrimonio = request.form['patrimonio']
        nome = request.form['nome']
        quantidade = request.form['quantidade']

        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute('''
        SELECT * FROM ferramentas WHERE patrimonio = ?
        ''', (patrimonio,))
        if cursor.fetchone() is not None:
            flash('Ferramenta já cadastrada na base de dados!')
            conn.close()
            return redirect(url_for('index'))

        cursor.execute('''
        INSERT INTO ferramentas (patrimonio, nome, quantidade)
        VALUES (?, ?, ?)
        ''', (patrimonio, nome, quantidade))
        conn.commit()
        conn.close()
        flash('Ferramenta adicionada com sucesso!')
        return redirect(url_for('index'))
    return render_template('adicionar.html')


# Rota para remover ferramenta
@app.route('/remover', methods=('POST',))
def remover():
    patrimonio = request.form['patrimonio_remover']

    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute('''
    DELETE FROM ferramentas WHERE patrimonio = ?
    ''', (patrimonio,))
    conn.commit()
    conn.close()
    flash('Ferramenta removida com sucesso!')
    return redirect(url_for('index'))


# Rota para registrar obra
@app.route('/registrar', methods=('GET', 'POST'))
def registrar():
    if request.method == 'POST':
        patrimonio = request.form['patrimonio']
        obra = request.form['obra']

        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute('''
        INSERT INTO registros (patrimonio, obra)
        VALUES (?, ?)
        ''', (patrimonio, obra))
        conn.commit()
        conn.close()
        flash('Registro de obra adicionado com sucesso!')
        return redirect(url_for('index'))
    return render_template('registrar.html')


if __name__ == '__main__':
    app.run(debug=True)
