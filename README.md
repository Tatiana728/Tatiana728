import customtkinter as ctk
from customtkinter import CTk
import tkinter as tk
from tkinter import ttk
from tkinter import messagebox
import mysql.connector
from mysql.connector import Error
from datetime import datetime
import csv

def setup_database():
    connection = None
    try:
        connection = mysql.connector.connect(
            host='localhost',
            user='root',
            password='acesso123',
            database='dietas_usp'
        )
        if connection.is_connected():
            cursor = connection.cursor()
            cursor.execute("DROP TABLE IF EXISTS usuario")
            cursor.execute("DROP TABLE IF EXISTS dieta")
            cursor.execute("DROP TABLE IF EXISTS alimento")
            cursor.execute("DROP TABLE IF EXISTS refeicao")
            cursor.execute("DROP TABLE IF EXISTS refeicao_alimento")
            cursor.execute("DROP TABLE IF EXISTS paciente")
            cursor.execute("DROP TABLE IF EXISTS paciente_dieta")
            cursor.execute("DROP TABLE IF EXISTS relatorio_dietas")
            cursor.execute("DROP TABLE IF EXISTS historico")  # Drop historico if it exists

            # Tabela de profissionais de saúde
            cursor.execute('''CREATE TABLE usuario (
                id INT AUTO_INCREMENT PRIMARY KEY,
                nome VARCHAR(255),
                email VARCHAR(255),
                tipo ENUM('nutrólogo','nutricionista'),
                permissao ENUM('administrador','usuario')
            )''')

            cursor.execute('''CREATE TABLE dieta (
                id INT AUTO_INCREMENT PRIMARY KEY,
                nome VARCHAR(255),
                calorias_diarias INT,
                calorias_totais INT
            )''')

            cursor.execute('''CREATE TABLE alimento (
                id INT AUTO_INCREMENT PRIMARY KEY,
                nome VARCHAR(255),
                calorias INT,
                proteinas FLOAT,
                carboidratos FLOAT,
                gorduras FLOAT
            )''')

            cursor.execute('''CREATE TABLE refeicao (
                id INT AUTO_INCREMENT PRIMARY KEY,
                nome VARCHAR(255),
                horario TIME,
                id_dieta INT,
                FOREIGN KEY (id_dieta) REFERENCES dieta (id)
            )''')

            cursor.execute('''CREATE TABLE refeicao_alimento (
                id_refeicao INT,
                id_alimento INT,
                quantidade FLOAT,
                FOREIGN KEY (id_refeicao) REFERENCES refeicao (id),
                FOREIGN KEY (id_alimento) REFERENCES alimento (id)
            )''')

            cursor.execute('''CREATE TABLE paciente (
                id INT AUTO_INCREMENT PRIMARY KEY,
                nome VARCHAR(255),
                email VARCHAR(255),
                id_profissional INT,
                FOREIGN KEY (id_profissional) REFERENCES usuario (id)
            )''')

            cursor.execute('''CREATE TABLE paciente_dieta (
                id_paciente INT,
                id_dieta INT,
                FOREIGN KEY (id_paciente) REFERENCES paciente (id),
                FOREIGN KEY (id_dieta) REFERENCES dieta (id)
            )''')

            cursor.execute('''CREATE TABLE relatorio_dietas (
                id INT AUTO_INCREMENT PRIMARY KEY,
                id_paciente INT,
                consumo_diario INT,
                consumo_semanal INT,
                FOREIGN KEY (id_paciente) REFERENCES paciente (id)
            )''')

            # Tabela de histórico de ações
            cursor.execute('''CREATE TABLE historico (
                id INT AUTO_INCREMENT PRIMARY KEY,
                acao VARCHAR(255),
                usuario VARCHAR(255),
                data_hora DATETIME
            )''')

            connection.commit()
    except Error as e:
        print(f"Error connecting to MySQL: {e}")
    finally:
        if connection and connection.is_connected():
            cursor.close()
            connection.close()

def log_action(acao, usuario="root"):
    try:
        connection = mysql.connector.connect(
            host='localhost',
            user='root',
            password='acesso123',
            database='dietas_usp'
        )
        cursor = connection.cursor()
        data_hora = datetime.now()
        cursor.execute("INSERT INTO historico (acao, usuario, data_hora) VALUES (%s, %s, %s)", (acao, usuario, data_hora))
        connection.commit()
    except Error as e:
        print(f"Error logging action: {e}")
    finally:
        if connection and connection.is_connected():
            cursor.close()
            connection.close()

# Função de cadastro de profissionais
def add_usuario(nome, email, tipo, permissao):
    try:
        connection = mysql.connector.connect(
            host='localhost',
            user='root',
            password='acesso123',
            database='dietas_usp'
        )
        cursor = connection.cursor()
        cursor.execute("INSERT INTO usuario (nome, email, tipo, permissao) VALUES (%s, %s, %s, %s)", (nome, email, tipo, permissao))
        connection.commit()
        log_action(f"Usuário '{nome}' cadastrado", "root")
        messagebox.showinfo("Success", "Profissional cadastrado com sucesso!")
    except Error as e:
        messagebox.showerror("Error", f"Erro ao cadastrar profissional: {e}")
    finally:
        if connection and connection.is_connected():
            cursor.close()
            connection.close()

# Função de cadastro de pacientes
def add_paciente(nome, email, id_profissional):
    try:
        connection = mysql.connector.connect(
            host='localhost',
            user='root',
            password='acesso123',
            database='dietas_usp'
        )
        cursor = connection.cursor()
        cursor.execute("INSERT INTO paciente (nome, email, id_profissional) VALUES (%s, %s, %s)", (nome, email, id_profissional))
        connection.commit()
        log_action(f"Paciente '{nome}' cadastrado", "root")
        messagebox.showinfo("Success", "Paciente cadastrado com sucesso!")
    except Error as e:
        messagebox.showerror("Error", f"Erro ao cadastrar paciente: {e}")
    finally:
        if connection and connection.is_connected():
            cursor.close()
            connection.close()

# Função de exportação de dietas
def exportar_dieta(id_paciente):
    try:
        connection = mysql.connector.connect(
            host='localhost',
            user='root',
            password='acesso123',
            database='dietas_usp'
        )
        cursor = connection.cursor()

        # Recuperar informações do paciente e dieta
        cursor.execute('''SELECT p.nome, d.nome, d.calorias_totais
                          FROM paciente_dieta pd
                          JOIN paciente p ON p.id = pd.id_paciente
                          JOIN dieta d ON d.id = pd.id_dieta
                          WHERE pd.id_paciente = %s''', (id_paciente,))
        dados_dieta = cursor.fetchone()

        if not dados_dieta:
            messagebox.showerror("Error", "Paciente ou dieta não encontrados")
            return

        nome_paciente, nome_dieta, calorias_totais = dados_dieta

        # Exportar para CSV
        with open(f'dieta_{nome_paciente}.csv', mode='w', newline='') as file:
            writer = csv.writer(file)
            writer.writerow(['Paciente', 'Dieta', 'Calorias Totais'])
            writer.writerow([nome_paciente, nome_dieta, calorias_totais])

        log_action(f"Dieta do paciente '{nome_paciente}' exportada")
        messagebox.showinfo("Success", "Dieta exportada com sucesso!")
    except Error as e:
        messagebox.showerror("Error", f"Erro ao exportar dieta: {e}")
    finally:
        if connection and connection.is_connected():
            cursor.close()
            connection.close()

# Login Window
def login_window():    
    def login():
        username = entry_user.get()
        password = entry_pass.get()
        if username == "root" and password == "acesso123":
            messagebox.showinfo("Login", "Login Bem-Sucedido!")
            window.destroy()
            app = DietaApp()
            app.mainloop()
        else:
            messagebox.showerror("Login", "Usuário ou senha inválidos.")
 
    window = ctk.CTk()
    window.title("Tela de Login")
    window.geometry("400x200")
    ctk.CTkLabel(window, text="Usuário:").pack()
    entry_user = ctk.CTkEntry(window)
    entry_user.pack()
    ctk.CTkLabel(window, text="Senha:").pack()
    entry_pass = ctk.CTkEntry(window, show='*')
    entry_pass.pack()
    ctk.CTkButton(window, text="Login", command=login).pack()
    window.mainloop()
 
if __name__ == "__main__":
    setup_database()
    login_window()
 
# Funções de GUI e fluxo de gestão
class DietaApp(ctk.CTk):
    def __init__(self):
        super().__init__()
        self.title("GERENCIAMENTO DE DIETAS")
        self.geometry("700x500")
       
        # Interface tabulada
        self.tab_control = ctk.CTkTabview(self)
        self.tab_control.pack(expand=True, fill='both')

        # Criação de abas
        self.create_profissional_tab()
        self.create_paciente_tab()
        self.create_dieta_tab()
        self.create_alimento_tab()
        self.create_refeicao_tab()
        self.create_historico_tab()

    def create_profissional_tab(self):
        prof_tab = self.tab_control.add("Profissionais")
        ctk.CTkLabel(prof_tab, text="Cadastro de Profissionais").pack()
        self.nome_prof_entry = ctk.CTkEntry(prof_tab, placeholder_text="Nome do Profissional")
        self.nome_prof_entry.pack()
        self.email_prof_entry = ctk.CTkEntry(prof_tab, placeholder_text="Email do Profissional")
        self.email_prof_entry.pack()
        self.tipo_prof_entry = ctk.CTkEntry(prof_tab, placeholder_text="Tipo (nutrólogo, nutricionista)")
        self.tipo_prof_entry.pack()
        self.permissao_prof_entry = ctk.CTkEntry(prof_tab, placeholder_text="Permissão (administrador, usuario)")
        self.permissao_prof_entry.pack()
        self.btn_add_prof = ctk.CTkButton(prof_tab, text="Adicionar Profissional", command=self.cadastrar_profissional)
        self.btn_add_prof.pack()

    def create_paciente_tab(self):
        paciente_tab = self.tab_control.add("Pacientes")
        ctk.CTkLabel(paciente_tab, text="Cadastro de Pacientes").pack()
        self.nome_paciente_entry = ctk.CTkEntry(paciente_tab, placeholder_text="Nome do Paciente")
        self.nome_paciente_entry.pack()
        self.email_paciente_entry = ctk.CTkEntry(paciente_tab, placeholder_text="Email do Paciente")
        self.email_paciente_entry.pack()
        self.id_profissional_entry = ctk.CTkEntry(paciente_tab, placeholder_text="ID do Profissional Responsável")
        self.id_profissional_entry.pack()
        self.btn_add_paciente = ctk.CTkButton(paciente_tab, text="Adicionar Paciente", command=self.cadastrar_paciente)
        self.btn_add_paciente.pack()

    def create_dieta_tab(self):
        dieta_tab = self.tab_control.add("Dietas")
        ctk.CTkLabel(dieta_tab, text="Cadastro de Dietas").pack()
        self.nome_dieta_entry = ctk.CTkEntry(dieta_tab, placeholder_text="Nome da Dieta")
        self.nome_dieta_entry.pack()
        self.calorias_diaria_entry = ctk.CTkEntry(dieta_tab, placeholder_text="Calorias Diárias")
        self.calorias_diaria_entry.pack()
        self.calorias_totais_entry = ctk.CTkEntry(dieta_tab, placeholder_text="Calorias Totais")
        self.calorias_totais_entry.pack()
        self.btn_add_dieta = ctk.CTkButton(dieta_tab, text="Adicionar Dieta", command=self.cadastrar_dieta)
        self.btn_add_dieta.pack()

    def create_alimento_tab(self):
        alimento_tab = self.tab_control.add("Alimentos")
        ctk.CTkLabel(alimento_tab, text="Cadastro de Alimentos").pack()
        self.nome_alimento_entry = ctk.CTkEntry(alimento_tab, placeholder_text="Nome do Alimento")
        self.nome_alimento_entry.pack()
        self.calorias_entry = ctk.CTkEntry(alimento_tab, placeholder_text="Calorias")
        self.calorias_entry.pack()
        self.proteinas_entry = ctk.CTkEntry(alimento_tab, placeholder_text="Proteínas")
        self.proteinas_entry.pack()
        self.carboidratos_entry = ctk.CTkEntry(alimento_tab, placeholder_text="Carboidratos")
        self.carboidratos_entry.pack()
        self.gorduras_entry = ctk.CTkEntry(alimento_tab, placeholder_text="Gorduras")
        self.gorduras_entry.pack()
        self.btn_add_alimento = ctk.CTkButton(alimento_tab, text="Adicionar Alimento", command=self.cadastrar_alimento)
        self.btn_add_alimento.pack()

    def create_refeicao_tab(self):
        refeicao_tab = self.tab_control.add("Refeições")
        ctk.CTkLabel(refeicao_tab, text="Cadastro de Refeições").pack()
        self.nome_refeicao_entry = ctk.CTkEntry(refeicao_tab, placeholder_text="Nome da Refeição")
        self.nome_refeicao_entry.pack()
        self.horario_refeicao_entry = ctk.CTkEntry(refeicao_tab, placeholder_text="Horário (HH:MM)")
        self.horario_refeicao_entry.pack()
        self.id_dieta_entry = ctk.CTkEntry(refeicao_tab, placeholder_text="ID da Dieta")
        self.id_dieta_entry.pack()
        self.btn_add_refeicao = ctk.CTkButton(refeicao_tab, text="Adicionar Refeição", command=self.cadastrar_refeicao)
        self.btn_add_refeicao.pack()

    def create_historico_tab(self):
        historico_tab = self.tab_control.add("Histórico")
        ctk.CTkButton(historico_tab, text="Mostrar Histórico", command=self.mostrar_historico).pack()

    def cadastrar_profissional(self):
        add_usuario(self.nome_prof_entry.get(), self.email_prof_entry.get(), self.tipo_prof_entry.get(), self.permissao_prof_entry.get())

    def cadastrar_paciente(self):
        add_paciente(self.nome_paciente_entry.get(), self.email_paciente_entry.get(), self.id_profissional_entry.get())

    def cadastrar_dieta(self):
        add_dieta(self.nome_dieta_entry.get(), self.calorias_totais_entry.get(), self.calorias_diaria_entry.get())

    def cadastrar_alimento(self):
        add_alimento(self.nome_alimento_entry.get(), self.calorias_entry.get(), self.proteinas_entry.get(), self.carboidratos_entry.get(), self.gorduras_entry.get())

    def cadastrar_refeicao(self):
        add_refeicao(self.nome_refeicao_entry.get(), self.horario_refeicao_entry.get(), self.id_dieta_entry.get())

    def mostrar_historico(self):
        connection = None
        cursor = None
        try:
            connection = mysql.connector.connect(
                host='localhost',
                user='root',
                password='acesso123',
                database='dietas_usp'
            )
            cursor = connection.cursor()
            cursor.execute("SELECT * FROM historico ORDER BY data_hora DESC")
            rows = cursor.fetchall()
           
            historico_window = ctk.CTkToplevel()
            historico_window.title("Histórico de Ações")
           
            tree = ttk.Treeview(historico_window, columns=("ID", "Ação", "Usuário", "Data/Hora"), show="headings")
            for col in tree["columns"]:
                tree.heading(col, text=col)
                tree.column(col, anchor='center')

            for row in rows:
                tree.insert("", "end", values=row)

            tree.pack(expand=True, fill="both")
            ctk.CTkButton(historico_window, text="Fechar", command=historico_window.destroy).pack(pady=10)

        except Error as e:
            messagebox.showerror("Erro", f"Erro ao mostrar histórico: {e}")
        finally:
            if cursor:
                cursor.close()
            if connection and connection.is_connected():
                connection.close()

# Função principal
if __name__ == "__main__":
    setup_database()
    app = DietaApp()
    app.mainloop()

 
