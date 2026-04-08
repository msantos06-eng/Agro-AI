/agro-ai


import streamlit as st
import numpy as np
import matplotlib.pyplot as plt
import rasterio
import tempfile
import os

from auth import login, cadastrar
from db import get_connection

st.set_page_config(page_title="Agro AI", layout="wide")

# ---------------- LOGIN ----------------
if "user" not in st.session_state:
    st.session_state.user = None

def tela_login():
    st.title("🌾 Agro AI - Login")

    opcao = st.radio("Escolha", ["Login", "Cadastro"])

    username = st.text_input("Usuário")
    password = st.text_input("Senha", type="password")

    if opcao == "Cadastro":
        if st.button("Cadastrar"):
            if cadastrar(username, password):
                st.success("Usuário criado!")
            else:
                st.error("Usuário já existe")

    else:
        if st.button("Entrar"):
            user = login(username, password)
            if user:
                st.session_state.user = user
                st.rerun()
            else:
                st.error("Login inválido")

if st.session_state.user is None:
    tela_login()
    st.stop()

# ---------------- APP ----------------

st.sidebar.title("Menu")
menu = st.sidebar.radio("Navegação", ["Fazendas", "NDVI"])

conn = get_connection()
c = conn.cursor()

# ---------------- FAZENDAS ----------------
if menu == "Fazendas":
    st.title("🚜 Gestão de Fazendas")

    nome_fazenda = st.text_input("Nome da Fazenda")

    if st.button("Adicionar Fazenda"):
        c.execute("INSERT INTO fazendas (user_id, nome) VALUES (?, ?)", (st.session_state.user[0], nome_fazenda))
        conn.commit()
        st.success("Fazenda adicionada")

    st.subheader("Minhas Fazendas")
    c.execute("SELECT * FROM fazendas WHERE user_id=?", (st.session_state.user[0],))
    fazendas = c.fetchall()

    for f in fazendas:
        st.write(f"📍 {f[2]}")

# ---------------- NDVI ----------------
if menu == "NDVI":
    st.title("🛰️ Análise NDVI")

    arquivo = st.file_uploader("Envie imagem multibanda (GeoTIFF)", type=["tif"])

    if arquivo:
        with tempfile.NamedTemporaryFile(delete=False) as tmp:
            tmp.write(arquivo.read())
            path = tmp.name

        with rasterio.open(path) as src:
            red = src.read(3).astype(float)
            nir = src.read(4).astype(float)

            ndvi = (nir - red) / (nir + red + 1e-5)

            fig, ax = plt.subplots()
            img = ax.imshow(ndvi, cmap="RdYlGn")
            plt.colorbar(img)
            st.pyplot(fig)

        os.remove(path)

conn.close()
from db import get_connection, init_db

init_db()

def cadastrar(username, password):
    conn = get_connection()
    c = conn.cursor()

    try:
        c.execute("INSERT INTO users (username, password) VALUES (?, ?)", (username, password))
        conn.commit()
        return True
    except:
        return False
    finally:
        conn.close()

def login(username, password):
    conn = get_connection()
    c = conn.cursor()

    c.execute("SELECT * FROM users WHERE username=? AND password=?", (username, password))
    user = c.fetchone()

    conn.close()
    return user
    import sqlite3

DB_NAME = "database.db"

def get_connection():
    return sqlite3.connect(DB_NAME, check_same_thread=False)

def init_db():
    conn = get_connection()
    c = conn.cursor()

    c.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE,
        password TEXT
    )
    """)

    c.execute("""
    CREATE TABLE IF NOT EXISTS fazendas (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        nome TEXT
    )
    """)

    c.execute("""
    CREATE TABLE IF NOT EXISTS talhoes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        fazenda_id INTEGER,
        nome TEXT
    )
    """)

    conn.commit()
    conn.close()
    streamlit
numpy
matplotlib
rasterio
geopandas
shapely
fiona
pyproj
pandas
#!/bin/bash

echo "Iniciando Agro AI..."

streamlit run app.py --server.port $PORT --server.address 0.0.0.0
