import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import sqlite3
import hashlib
import os

class MessagingApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Application de Messagerie")
        self.root.geometry("900x600")
        self.root.resizable(False, False)
        
        self.current_user = None
        self.current_chat = None
        self.db_path = 'messaging_shared.db'
        self.credentials_file = 'saved_credentials.txt'
        
        self.init_database()
        
        if self.load_saved_credentials():
            self.show_main_screen()
        else:
            self.show_login_screen()
    
    def init_database(self):
        self.conn = sqlite3.connect(self.db_path)
        self.cursor = self.conn.cursor()
        
        self.cursor.execute('''CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            pseudo TEXT UNIQUE NOT NULL,
            password TEXT NOT NULL,
            user_id TEXT UNIQUE NOT NULL,
            email TEXT,
            profile_pic TEXT,
            bg_color TEXT DEFAULT '#FF6B9D'
        )''')
        
        self.cursor.execute('''CREATE TABLE IF NOT EXISTS messages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            sender_id INTEGER,
            receiver_id INTEGER,
            message TEXT,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (sender_id) REFERENCES users (id),
            FOREIGN KEY (receiver_id) REFERENCES users (id)
        )''')
        
        self.cursor.execute('''CREATE TABLE IF NOT EXISTS groups (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            creator_id INTEGER,
            FOREIGN KEY (creator_id) REFERENCES users (id)
        )''')
        
        self.cursor.execute('''CREATE TABLE IF NOT EXISTS group_members (
            group_id INTEGER,
            user_id INTEGER,
            FOREIGN KEY (group_id) REFERENCES groups (id),
            FOREIGN KEY (user_id) REFERENCES users (id)
        )''')
        
        self.cursor.execute('''CREATE TABLE IF NOT EXISTS group_messages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            group_id INTEGER,
            sender_id INTEGER,
            message TEXT,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (group_id) REFERENCES groups (id),
            FOREIGN KEY (sender_id) REFERENCES users (id)
        )''')
        
        self.conn.commit()
    
    def hash_password(self, password):
        return hashlib.sha256(password.encode()).hexdigest()
    
    def generate_user_id(self):
        import random, string
        while True:
            user_id = ''.join(random.choices(string.ascii_uppercase + string.digits, k=8))
            self.cursor.execute('SELECT id FROM users WHERE user_id = ?', (user_id,))
            if not self.cursor.fetchone():
                return user_id
    
    def save_credentials(self, pseudo, password):
        try:
            with open(self.credentials_file, 'w') as f:
                f.write(f"{pseudo}\n{password}")
        except: pass
    
    def load_saved_credentials(self):
        try:
            if os.path.exists(self.credentials_file):
                with open(self.credentials_file, 'r') as f:
                    lines = f.readlines()
                    if len(lines) >= 2:
                        pseudo, password = lines[0].strip(), lines[1].strip()
                        hashed_pw = self.hash_password(password)
                        self.cursor.execute('SELECT * FROM users WHERE pseudo = ? AND password = ?', (pseudo, hashed_pw))
                        user = self.cursor.fetchone()
                        if user:
                            self.current_user = {'id': user[0], 'pseudo': user[1], 'user_id': user[3], 'email': user[4], 'profile_pic': user[5], 'bg_color': user[6]}
                            return True
        except: pass
        return False
    
    def logout(self):
        try:
            if os.path.exists(self.credentials_file):
                os.remove(self.credentials_file)
        except: pass
        self.current_user = None
        self.show_login_screen()
    
    def clear_screen(self):
        for widget in self.root.winfo_children():
            widget.destroy()
    
    def show_login_screen(self):
        self.clear_screen()
        self.root.configure(bg='#FF6B6B')
        
        tk.Label(self.root, text="BIENVENUE", font=("Arial", 48, "bold"), fg="white", bg='#FF6B6B').pack(pady=40)
        
        frame = tk.Frame(self.root, bg='#8B4513', bd=0)
        frame.pack(pady=50)
        
        tk.Label(frame, text="pseudo", font=("Arial", 14), bg='#8B4513', fg='white').pack(pady=(30, 5))
        self.login_pseudo = tk.Entry(frame, font=("Arial", 14), bg='#654321', fg='white', width=30)
        self.login_pseudo.pack(pady=5, padx=40)
        
        tk.Label(frame, text="mot de passe", font=("Arial", 14), bg='#8B4513', fg='white').pack(pady=(15, 5))
        self.login_password = tk.Entry(frame, font=("Arial", 14), show='*', bg='#654321', fg='white', width=30)
        self.login_password.pack(pady=5, padx=40)
        
        btn_frame = tk.Frame(frame, bg='#8B4513')
        btn_frame.pack(pady=30)
        
        tk.Button(btn_frame, text="Connexion", font=("Arial", 12), bg='#4CAF50', fg='white', command=self.login, width=12).pack(side='left', padx=5)
        tk.Button(btn_frame, text="Inscription", font=("Arial", 12), bg='#2196F3', fg='white', command=self.show_register_screen, width=12).pack(side='left', padx=5)
    
    def show_register_screen(self):
        self.clear_screen()
        self.root.configure(bg='#FF6B6B')
        
        tk.Label(self.root, text="INSCRIPTION", font=("Arial", 36, "bold"), fg="white", bg='#FF6B6B').pack(pady=30)
        
        frame = tk.Frame(self.root, bg='#8B4513')
        frame.pack(pady=30)
        
        tk.Label(frame, text="Pseudo", font=("Arial", 12), bg='#8B4513', fg='white').pack(pady=(20, 2))
        self.reg_pseudo = tk.Entry(frame, font=("Arial", 12), bg='#654321', fg='white', width=30)
        self.reg_pseudo.pack(pady=2, padx=40)
        
        tk.Label(frame, text="Mot de passe", font=("Arial", 12), bg='#8B4513', fg='white').pack(pady=(10, 2))
        self.reg_password = tk.Entry(frame, font=("Arial", 12), show='*', bg='#654321', fg='white', width=30)
        self.reg_password.pack(pady=2, padx=40)
        
        tk.Label(frame, text="Email (optionnel)", font=("Arial", 12), bg='#8B4513', fg='white').pack(pady=(10, 2))
        self.reg_email = tk.Entry(frame, font=("Arial", 12), bg='#654321', fg='white', width=30)
        self.reg_email.pack(pady=2, padx=40)
        
        btn_frame = tk.Frame(frame, bg='#8B4513')
        btn_frame.pack(pady=25)
        
        tk.Button(btn_frame, text="S'inscrire", font=("Arial", 12), bg='#4CAF50', fg='white', command=self.register, width=12).pack(side='left', padx=5)
        tk.Button(btn_frame, text="Retour", font=("Arial", 12), bg='#f44336', fg='white', command=self.show_login_screen, width=12).pack(side='left', padx=5)
    
    def register(self):
        pseudo = self.reg_pseudo.get().strip()
        password = self.reg_password.get()
        email = self.reg_email.get().strip()
        
        if not pseudo or not password:
            messagebox.showerror("Erreur", "Pseudo et mot de passe requis!")
            return
        
        self.cursor.execute('SELECT id FROM users WHERE pseudo = ?', (pseudo,))
        if self.cursor.fetchone():
            messagebox.showerror("Erreur", "Ce pseudo est déjà pris!")
            return
        
        try:
            hashed_pw = self.hash_password(password)
            user_id = self.generate_user_id()
            self.cursor.execute('INSERT INTO users (pseudo, password, user_id, email) VALUES (?, ?, ?, ?)', (pseudo, hashed_pw, user_id, email))
            self.conn.commit()
            messagebox.showinfo("Succès", f"Compte créé!\n\nVotre ID: {user_id}\n\nNotez-le bien!")
            self.show_login_screen()
        except Exception as e:
            messagebox.showerror("Erreur", f"Erreur: {e}")
    
    def login(self):
        pseudo = self.login_pseudo.get().strip()
        password = self.login_password.get()
        
        if not pseudo or not password:
            messagebox.showerror("Erreur", "Remplissez tous les champs!")
            return
        
        hashed_pw = self.hash_password(password)
        self.cursor.execute('SELECT * FROM users WHERE pseudo = ? AND password = ?', (pseudo, hashed_pw))
        user = self.cursor.fetchone()
        
        if user:
            self.current_user = {'id': user[0], 'pseudo': user[1], 'user_id': user[3], 'email': user[4], 'profile_pic': user[5], 'bg_color': user[6]}
            self.save_credentials(pseudo, password)
            self.show_main_screen()
        else:
            messagebox.showerror("Erreur", "Identifiants incorrects!")
    
    def show_main_screen(self):
        self.clear_screen()
        try:
            self.root.configure(bg=self.current_user['bg_color'])
        except:
            self.root.configure(bg='#FF6B9D')
        
        header = tk.Frame(self.root, bg='#1E1E1E', height=60)
        header.pack(fill='x', pady=(10, 5), padx=10)
        
        profile_label = tk.Label(header, text="👤", font=("Arial", 25), bg='#1E1E1E')
        profile_label.pack(side='left', padx=10)
        
        try:
            if self.current_user.get('profile_pic') and os.path.exists(self.current_user['profile_pic']):
                from PIL import Image, ImageTk
                img = Image.open(self.current_user['profile_pic'])
                img = img.resize((45, 45), Image.Resampling.LANCZOS)
                photo = ImageTk.PhotoImage(img)
                profile_label.config(image=photo, text="")
                profile_label.image = photo
        except:
            pass
        
        tk.Label(header, text=self.current_user['pseudo'], font=("Arial", 16, "bold"), bg='#1E1E1E', fg='white').pack(side='left', padx=5)
        tk.Label(header, text=f"ID: {self.current_user['user_id']}", font=("Arial", 10), bg='#1E1E1E', fg='#4CAF50').pack(side='left', padx=5)
        
        main_frame = tk.Frame(self.root)
        try:
            main_frame.configure(bg=self.current_user['bg_color'])
        except:
            main_frame.configure(bg='#FF6B9D')
        main_frame.pack(fill='both', expand=True, padx=10)
        
        left = tk.Frame(main_frame, bg='#2C2C2C')
        left.pack(side='left', fill='both', expand=True, padx=(0, 5))
        
        tk.Label(left, text="CHATS", font=("Arial", 18, "bold"), bg='#2C2C2C', fg='white').pack(pady=10)
        self.chat_listbox = tk.Listbox(left, font=("Arial", 12), bg='#3C3C3C', fg='white', selectbackground='#555555')
        self.chat_listbox.pack(fill='both', expand=True, padx=10, pady=5)
        self.chat_listbox.bind('<<ListboxSelect>>', self.open_chat)
        tk.Button(left, text="AJOUTER PERSONNE", font=("Arial", 11, "bold"), bg='#4A4A4A', fg='white', command=self.add_person).pack(pady=5)
        
        right = tk.Frame(main_frame, bg='#2C2C2C')
        right.pack(side='right', fill='both', expand=True, padx=(5, 0))
        
        tk.Label(right, text="GROUPES", font=("Arial", 18, "bold"), bg='#2C2C2C', fg='white').pack(pady=10)
        self.group_listbox = tk.Listbox(right, font=("Arial", 12), bg='#3C3C3C', fg='white', selectbackground='#555555')
        self.group_listbox.pack(fill='both', expand=True, padx=10, pady=5)
        self.group_listbox.bind('<<ListboxSelect>>', self.open_group_chat)
        tk.Button(right, text="CRÉER GROUPE", font=("Arial", 11, "bold"), bg='#4A4A4A', fg='white', command=self.create_group).pack(pady=5)
        
        bottom = tk.Frame(self.root)
        try:
            bottom.configure(bg=self.current_user['bg_color'])
        except:
            bottom.configure(bg='#FF6B9D')
        bottom.pack(fill='x', pady=10)
        tk.Button(bottom, text="PARAMÈTRES", font=("Arial", 12, "bold"), bg='#1E1E1E', fg='white', command=self.show_settings, width=20).pack(side='left', padx=10)
        tk.Button(bottom, text="Déconnexion", font=("Arial", 10), bg='#E74C3C', fg='white', command=self.logout, width=15).pack(side='right', padx=10)
        
        self.refresh_user_list()
        self.refresh_group_list()
    
    def refresh_user_list(self):
        self.chat_listbox.delete(0, tk.END)
        self.cursor.execute('SELECT id, pseudo, user_id FROM users WHERE id != ?', (self.current_user['id'],))
        for user in self.cursor.fetchall():
            self.chat_listbox.insert(tk.END, f"{user[1]} (ID: {user[2]})")
    
    def refresh_group_list(self):
        self.group_listbox.delete(0, tk.END)
        self.cursor.execute('''SELECT DISTINCT g.id, g.name FROM groups g
            JOIN group_members gm ON g.id = gm.group_id WHERE gm.user_id = ?''', (self.current_user['id'],))
        for group in self.cursor.fetchall():
            self.group_listbox.insert(tk.END, f"{group[1]} (groupe)")
    
    def add_person(self):
        dialog = tk.Toplevel(self.root)
        dialog.title("Ajouter une personne")
        dialog.geometry("400x200")
        dialog.configure(bg='#2C2C2C')
        
        tk.Label(dialog, text="ID de la personne:", font=("Arial", 12), bg='#2C2C2C', fg='white').pack(pady=20)
        id_entry = tk.Entry(dialog, font=("Arial", 12), bg='#3C3C3C', fg='white')
        id_entry.pack(pady=10, padx=20, fill='x')
        tk.Label(dialog, text=f"Votre ID: {self.current_user['user_id']}", font=("Arial", 10), bg='#2C2C2C', fg='#4CAF50').pack(pady=5)
        
        def add():
            user_id = id_entry.get().strip().upper()
            if not user_id:
                return messagebox.showerror("Erreur", "Entrez un ID!")
            self.cursor.execute('SELECT id, pseudo FROM users WHERE user_id = ?', (user_id,))
            user = self.cursor.fetchone()
            if not user:
                return messagebox.showerror("Erreur", "ID inexistant!")
            if user[0] == self.current_user['id']:
                return messagebox.showerror("Erreur", "Vous ne pouvez pas vous ajouter!")
            messagebox.showinfo("Succès", f"{user[1]} ajouté!")
            dialog.destroy()
            self.refresh_user_list()
        
        tk.Button(dialog, text="Ajouter", font=("Arial", 12), bg='#4CAF50', fg='white', command=add).pack(pady=20)
    
    def open_chat(self, event):
        sel = self.chat_listbox.curselection()
        if not sel: return
        pseudo = self.chat_listbox.get(sel[0]).split(" (ID: ")[0]
        self.cursor.execute('SELECT id FROM users WHERE pseudo = ?', (pseudo,))
        user = self.cursor.fetchone()
        if user:
            self.current_chat = {'type': 'user', 'id': user[0], 'name': pseudo}
            self.show_chat_screen()
    
    def open_group_chat(self, event):
        sel = self.group_listbox.curselection()
        if not sel: return
        name = self.group_listbox.get(sel[0]).split(" (groupe)")[0]
        self.cursor.execute('SELECT id FROM groups WHERE name = ?', (name,))
        group = self.cursor.fetchone()
        if group:
            self.current_chat = {'type': 'group', 'id': group[0], 'name': name}
            self.show_chat_screen()
    
    def show_chat_screen(self):
        self.clear_screen()
        self.root.configure(bg='#2C2C2C')
        
        header = tk.Frame(self.root, bg='#1E1E1E', height=60)
        header.pack(fill='x')
        tk.Button(header, text="← Retour", font=("Arial", 12), bg='#333333', fg='white', command=self.show_main_screen).pack(side='left', padx=10)
        tk.Label(header, text=self.current_chat['name'], font=("Arial", 16, "bold"), bg='#1E1E1E', fg='white').pack(side='left', padx=20)
        
        self.messages_display = tk.Text(self.root, font=("Arial", 12), bg='#2C2C2C', fg='white', wrap='word', state='disabled')
        self.messages_display.pack(fill='both', expand=True, padx=50, pady=20)
        
        input_frame = tk.Frame(self.root, bg='#1E1E1E')
        input_frame.pack(fill='x', padx=50, pady=(0, 20))
        self.message_entry = tk.Entry(input_frame, font=("Arial", 14), bg='#3C3C3C', fg='white')
        self.message_entry.pack(side='left', fill='both', expand=True, padx=10, pady=10)
        self.message_entry.bind('<Return>', lambda e: self.send_message())
        tk.Button(input_frame, text="Envoyer", font=("Arial", 12, "bold"), bg='#4CAF50', fg='white', command=self.send_message, width=12).pack(side='right', padx=10, pady=10)
        
        self.load_messages()
        self.auto_refresh()
    
    def auto_refresh(self):
        if hasattr(self, 'messages_display') and self.messages_display.winfo_exists():
            self.load_messages()
            self.root.after(2000, self.auto_refresh)
    
    def load_messages(self):
        self.conn = sqlite3.connect(self.db_path)
        self.cursor = self.conn.cursor()
        self.messages_display.config(state='normal')
        self.messages_display.delete(1.0, tk.END)
        
        if self.current_chat['type'] == 'user':
            self.cursor.execute('''SELECT m.message, u.pseudo, m.sender_id FROM messages m
                JOIN users u ON m.sender_id = u.id
                WHERE (m.sender_id = ? AND m.receiver_id = ?) OR (m.sender_id = ? AND m.receiver_id = ?)
                ORDER BY m.timestamp''', (self.current_user['id'], self.current_chat['id'], self.current_chat['id'], self.current_user['id']))
        else:
            self.cursor.execute('''SELECT gm.message, u.pseudo, gm.sender_id FROM group_messages gm
                JOIN users u ON gm.sender_id = u.id WHERE gm.group_id = ? ORDER BY gm.timestamp''', (self.current_chat['id'],))
        
        for msg in self.cursor.fetchall():
            sender = "Moi" if msg[2] == self.current_user['id'] else msg[1]
            self.messages_display.insert(tk.END, f"{sender}: {msg[0]}\n\n")
        
        self.messages_display.config(state='disabled')
        self.messages_display.see(tk.END)
    
    def send_message(self):
        msg = self.message_entry.get().strip()
        if not msg: return
        
        if self.current_chat['type'] == 'user':
            self.cursor.execute('INSERT INTO messages (sender_id, receiver_id, message) VALUES (?, ?, ?)',
                              (self.current_user['id'], self.current_chat['id'], msg))
        else:
            self.cursor.execute('INSERT INTO group_messages (group_id, sender_id, message) VALUES (?, ?, ?)',
                              (self.current_chat['id'], self.current_user['id'], msg))
        self.conn.commit()
        self.message_entry.delete(0, tk.END)
        self.load_messages()
    
    def create_group(self):
        dialog = tk.Toplevel(self.root)
        dialog.title("Créer un groupe")
        dialog.geometry("400x180")
        dialog.configure(bg='#2C2C2C')
        
        tk.Label(dialog, text="Nom du groupe:", font=("Arial", 12), bg='#2C2C2C', fg='white').pack(pady=20)
        entry = tk.Entry(dialog, font=("Arial", 12), bg='#3C3C3C', fg='white')
        entry.pack(pady=10, padx=20, fill='x')
        
        def save():
            name = entry.get().strip()
            if not name:
                return messagebox.showerror("Erreur", "Entrez un nom!")
            self.cursor.execute('INSERT INTO groups (name, creator_id) VALUES (?, ?)', (name, self.current_user['id']))
            gid = self.cursor.lastrowid
            self.cursor.execute('INSERT INTO group_members (group_id, user_id) VALUES (?, ?)', (gid, self.current_user['id']))
            self.conn.commit()
            messagebox.showinfo("Succès", "Groupe créé!")
            dialog.destroy()
            self.refresh_group_list()
        
        tk.Button(dialog, text="Créer", font=("Arial", 12), bg='#4CAF50', fg='white', command=save).pack(pady=20)
    
    def show_settings(self):
        self.clear_screen()
        self.root.configure(bg='#1E90FF')
        
        frame = tk.Frame(self.root, bg='#2C3E50')
        frame.place(relx=0.5, rely=0.5, anchor='center', width=450, height=500)
        
        tk.Label(frame, text="Paramètres", font=("Arial", 22, "bold"), bg='#2C3E50', fg='white').pack(pady=10)
        
        tk.Label(frame, text="Pseudo", font=("Arial", 11), bg='#2C3E50', fg='white').pack(pady=(3, 1))
        pseudo_entry = tk.Entry(frame, font=("Arial", 11), bg='#34495E', fg='white')
        pseudo_entry.insert(0, self.current_user['pseudo'])
        pseudo_entry.pack(pady=1, padx=30, fill='x')
        
        tk.Label(frame, text=f"Votre ID: {self.current_user['user_id']}", font=("Arial", 9, "bold"), bg='#2C3E50', fg='#4CAF50').pack(pady=2)
        
        tk.Label(frame, text="Email", font=("Arial", 11), bg='#2C3E50', fg='white').pack(pady=(3, 1))
        email_entry = tk.Entry(frame, font=("Arial", 11), bg='#34495E', fg='white')
        email_entry.insert(0, self.current_user['email'] or '')
        email_entry.pack(pady=1, padx=30, fill='x')
        
        tk.Label(frame, text="Photo de profil", font=("Arial", 11), bg='#2C3E50', fg='white').pack(pady=(3, 1))
        tk.Button(frame, text="Changer la photo", font=("Arial", 9), bg='#34495E', fg='white', 
                 command=self.change_profile_pic).pack(pady=3)
        
        tk.Label(frame, text="Nouveau mot de passe", font=("Arial", 11), bg='#2C3E50', fg='white').pack(pady=(3, 1))
        pw_entry = tk.Entry(frame, font=("Arial", 11), show='*', bg='#34495E', fg='white')
        pw_entry.pack(pady=1, padx=30, fill='x')
        
        tk.Label(frame, text="Couleur de fond", font=("Arial", 11), bg='#2C3E50', fg='white').pack(pady=(3, 1))
        colors = ['#FF6B9D', '#11998e', '#C471ED', '#f7971e', '#667eea', '#000000', '#ff9a9e']
        color_var = tk.StringVar(value=self.current_user['bg_color'])
        color_menu = ttk.Combobox(frame, textvariable=color_var, values=colors, state='readonly', width=20)
        color_menu.pack(pady=3)
        
        def save():
            new_pseudo = pseudo_entry.get().strip()
            new_email = email_entry.get().strip()
            new_pw = pw_entry.get()
            new_color = color_var.get()
            
            if new_pw:
                hashed = self.hash_password(new_pw)
                self.cursor.execute('UPDATE users SET pseudo=?, email=?, password=?, bg_color=? WHERE id=?',
                                  (new_pseudo, new_email, hashed, new_color, self.current_user['id']))
            else:
                self.cursor.execute('UPDATE users SET pseudo=?, email=?, bg_color=? WHERE id=?',
                                  (new_pseudo, new_email, new_color, self.current_user['id']))
            
            self.conn.commit()
            self.current_user.update({'pseudo': new_pseudo, 'email': new_email, 'bg_color': new_color})
            messagebox.showinfo("Succès", "Paramètres sauvegardés!")
        
        btn_frame = tk.Frame(frame, bg='#2C3E50')
        btn_frame.pack(pady=12)
        tk.Button(btn_frame, text="Sauvegarder", font=("Arial", 10), bg='#27AE60', fg='white', command=save, width=11).pack(side='left', padx=3)
        tk.Button(btn_frame, text="Retour", font=("Arial", 10), bg='#3498DB', fg='white', command=self.show_main_screen, width=11).pack(side='left', padx=3)
        tk.Button(btn_frame, text="Déconnexion", font=("Arial", 10), bg='#E74C3C', fg='white', command=self.logout, width=11).pack(side='left', padx=3)
    
    def change_profile_pic(self):
        filepath = filedialog.askopenfilename(
            title="Choisir une photo",
            filetypes=[("Images", "*.png *.jpg *.jpeg *.gif *.bmp")]
        )
        if filepath:
            self.cursor.execute('UPDATE users SET profile_pic=? WHERE id=?', (filepath, self.current_user['id']))
            self.conn.commit()
            self.current_user['profile_pic'] = filepath
            messagebox.showinfo("Succès", "Photo mise à jour!")

if __name__ == "__main__":
    root = tk.Tk()
    app = MessagingApp(root)
    root.mainloop()
