# Agro-AI-pro
Ferramenta que auxilia o produtos nas suas tomadas de decisões.
# Agro AÍ

**Agro AÍ** é um aplicativo inovador que utiliza processamento de imagens de satélite e drones para ajudar agricultores nas tomadas de decisões estratégicas. Ele calcula índices como NDVI, realiza taxas variáveis e aplica tecnologias de agricultura de precisão, sendo o braço direito do produtor no campo.

## 🚀 Funcionalidades Principais
- 📷 **Processamento de Imagens** de satélite e drones para análise avançada do campo.
- 🌾 **Cálculo de NDVI** (Normalized Difference Vegetation Index) para monitorar saúde da vegetação.
- 📊 **Taxas Variáveis** para otimização no uso de insumos agrícolas.
- 🎯 **Aplicações Localizadas** que tornam decisões mais precisas e eficientes.

## 💻 Tecnologias Utilizadas
- **Processamento de Imagens**: Bibliotecas especializadas como OpenCV, GDAL ou alternativas.
- **Frontend**: Frameworks como React Native ou Flutter para uma interface intuitiva.
- **Backend**: Node.js, Django ou outra tecnologia de servidor para gerenciar os cálculos e dados.
- **Infraestrutura:** Serviços em nuvem como AWS, Google Cloud ou Azure.

## 📌 Status
🚧 Este projeto está em fase inicial de desenvolvimento.

## 🤝 Como Contribuir
Contribuições são sempre bem-vindas! Se você deseja contribuir com este projeto, siga os passos:
1. Faça um **fork** deste repositório.
2. Crie uma branch para sua feature (`git checkout -b minha-feature`).
3. Submeta suas alterações (`git commit -m 'Minha nova feature'`).
4. Envie sua branch para o repositório remoto (`git push origin minha-feature`).
5. Abra um **pull request**.

## 📜 Licença
Este projeto está licenciado sob a [MIT License](LICENSE).

---
import streamlit as st
import numpy as np
import rasterio
import matplotlib.pyplot as plt
import tempfile
import os
import sqlite3
import geopandas as gpd
from rasterio.features import shapes
from auth import criar_tabelas, cadastrar, login

st.set_page_config(page_title="Agro AI Pro", layout="wide")

criar_tabelas()

def conectar():
    return sqlite3.connect("database.db")

if "logado" not in st.session_state:
    st.session_state.logado = False

if not st.session_state.logado:

    st.title("🔐 Agro AI Pro")

    menu = st.radio("Escolha:", ["Login", "Cadastrar"])

    user = st.text_input("Usuário")
    senha = st.text_input("Senha", type="password")

    if menu == "Cadastrar":
        if st.button("Cadastrar"):
            cadastrar(user, senha)
            st.success("Usuário criado!")

    if menu == "Login":
        if st.button("Entrar"):
            if login(user, senha):
                st.session_state.logado = True
                st.session_state.usuario = user
                st.rerun()
            else:
                st.error("Login inválido")

else:

    usuario = st.session_state.usuario

    st.sidebar.write(f"👤 {usuario}")

    if st.sidebar.button("Sair"):
        st.session_state.logado = False
        st.rerun()

    conn = conectar()
    c = conn.cursor()

    st.sidebar.subheader("🏡 Fazendas")

    nova_fazenda = st.sidebar.text_input("Nova fazenda")

    if st.sidebar.button("Cadastrar fazenda"):
        c.execute("INSERT INTO fazendas (usuario, nome) VALUES (?, ?)",
                  (usuario, nova_fazenda))
        conn.commit()
        st.sidebar.success("Fazenda criada!")

    c.execute("SELECT nome FROM fazendas WHERE usuario=?", (usuario,))
    fazendas = [f[0] for f in c.fetchall()]

    fazenda_sel = st.sidebar.selectbox("Escolha a fazenda", fazendas)

    st.sidebar.subheader("🌾 Talhões")

    novo_talhao = st.sidebar.text_input("Novo talhão")

    if st.sidebar.button("Cadastrar talhão"):
        c.execute("INSERT INTO talhoes (fazenda, nome) VALUES (?, ?)",
                  (fazenda_sel, novo_talhao))
        conn.commit()
        st.sidebar.success("Talhão criado!")

    c.execute("SELECT nome FROM talhoes WHERE fazenda=?", (fazenda_sel,))
    talhoes = [t[0] for t in c.fetchall()]

    talhao_sel = st.sidebar.selectbox("Escolha o talhão", talhoes)

    conn.close()

    pasta = f"data/{usuario}/{fazenda_sel}/{talhao_sel}"
    os.makedirs(pasta, exist_ok=True)

    st.title("🌱 Agro AI Pro")

    nir_file = st.file_uploader("Banda NIR (.tif)", type=["tif"])
    red_file = st.file_uploader("Banda RED (.tif)", type=["tif"])

    if nir_file and red_file:

        with tempfile.NamedTemporaryFile(delete=False) as tmp_nir:
            tmp_nir.write(nir_file.read())
            nir_path = tmp_nir.name

        with tempfile.NamedTemporaryFile(delete=False) as tmp_red:
            tmp_red.write(red_file.read())
            red_path = tmp_red.name

        with rasterio.open(nir_path) as nir:
            nir_band = nir.read(1)
            meta = nir.meta

        with rasterio.open(red_path) as red:
            red_band = red.read(1)

        ndvi = (nir_band - red_band) / (nir_band + red_band + 0.001)

        st.success("NDVI gerado!")

        fig, ax = plt.subplots()
        cax = ax.imshow(ndvi, cmap='RdYlGn')
        fig.colorbar(cax)
        st.pyplot(fig)

        zonas = np.zeros_like(ndvi)
        zonas[ndvi < 0.3] = 1
        zonas[(ndvi >= 0.3) & (ndvi < 0.6)] = 2
        zonas[ndvi >= 0.6] = 3

        ndvi_path = f"{pasta}/ndvi.tif"
        meta.update(dtype=rasterio.float32)

        with rasterio.open(ndvi_path, 'w', **meta) as dst:
            dst.write(ndvi.astype(rasterio.float32), 1)

        if st.button("🚜 Gerar mapa de taxa variável"):

            results = (
                {"properties": {"zona": int(v)}, "geometry": s}
                for s, v in shapes(zonas.astype(np.int16), transform=meta["transform"])
            )

            gdf = gpd.GeoDataFrame.from_features(list(results))

            def taxa(z):
                if z == 1:
                    return 120
                elif z == 2:
                    return 90
                else:
                    return 60

            gdf["taxa_kg_ha"] = gdf["zona"].apply(taxa)

            shp_path = f"{pasta}/mapa.shp"
            gdf.to_file(shp_path)

            with open(shp_path, "rb") as f:
                st.download_button("📥 Baixar Shapefile", f, "mapa.shp")
import sqlite3
import hashlib

def conectar():
    return sqlite3.connect("database.db")

def criar_tabelas():
    conn = conectar()
    c = conn.cursor()

    c.execute("""
    CREATE TABLE IF NOT EXISTS usuarios (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT,
        password TEXT
    )
    """)

    c.execute("""
    CREATE TABLE IF NOT EXISTS fazendas (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        usuario TEXT,
        nome TEXT
    )
    """)

    c.execute("""
    CREATE TABLE IF NOT EXISTS talhoes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        fazenda TEXT,
        nome TEXT
    )
    """)

    conn.commit()
    conn.close()

def hash_senha(senha):
    return hashlib.sha256(senha.encode()).hexdigest()

def cadastrar(username, senha):
    conn = conectar()
    c = conn.cursor()
    c.execute("INSERT INTO usuarios (username, password) VALUES (?, ?)",
              (username, hash_senha(senha)))
    conn.commit()
    conn.close()

def login(username, senha):
    conn = conectar()
    c = conn.cursor()
    c.execute("SELECT * FROM usuarios WHERE username=? AND password=?",
              (username, hash_senha(senha)))
    user = c.fetchone()
    conn.close()
    return user
    streamlit
numpy
matplotlib
rasterio
geopandas
shapely
fiona
pyproj
streamlit run app.py --server.port $PORT --server.address 0.0.0.0
