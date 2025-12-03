# trabalho-redes
import socket
import threading

# Armazena clientes conectados: {username: socket}
clients = {}

# Broadcast para todos os clientes
def broadcast(message, ignore_user=None):
    for user, conn in clients.items():
        if user != ignore_user:
            try:
                conn.sendall((message + "\n").encode())
            except:
                pass

def handle_client(conn, addr):
    global clients
    username = None

    try:
        # -------- LOGIN --------
        data = conn.recv(1024).decode().strip()
        if not data.startswith("LOGIN|"):
            conn.sendall("LOGIN_FAIL|Formato inválido\n".encode())
            conn.close()
            return

        username = data.split("|")[1]

        if username in clients:
            conn.sendall("LOGIN_FAIL|Usuário já existe\n".encode())
            conn.close()
            return

        clients[username] = conn
        conn.sendall(f"LOGIN_OK|{username}\n".encode())

        # Notificar entrada
        broadcast(f"USER_JOINED|{username}", ignore_user=username)

        print(f"[+] {username} entrou ({addr})")

        # -------- LOOP PRINCIPAL --------
        while True:
            raw = conn.recv(4096)
            if not raw:
                break

            msg = raw.decode().strip()
            parts = msg.split("|")
            cmd = parts[0]

            # ------------------- LISTA -------------------
            if cmd == "LIST":
                user_list = ",".join(clients.keys())
                conn.sendall(f"LIST_OK|{user_list}\n".encode())

            # ------------------- MENSAGEM PRIVADA -------------------
            elif cmd == "MSG_TO":
                if len(parts) < 3:
                    continue
                dest = parts[1]
                text = parts[2]

                if dest not in clients:
                    conn.sendall(f"MSG_FAIL|{dest}|Usuário não encontrado\n".encode())
                else:
                    clients[dest].sendall(f"MSG_FROM|{username}|{text}\n".encode())
                    conn.sendall(f"MSG_SENT|{dest}\n".encode())

            # ------------------- BROADCAST -------------------
            elif cmd == "MSG_ALL":
                text = parts[1]
                broadcast(f"MSG_BROADCAST|{username}|{text}", ignore_user=None)

            # ------------------- LOGOUT -------------------
            elif cmd == "LOGOUT":
                conn.sendall("LOGOUT_OK\n".encode())
                break

    except ConnectionResetError:
        pass

    finally:
        # Usuário saiu
        if username in clients:
            del clients[username]
            broadcast(f"USER_LEFT|{username}")
            print(f"[-] {username} saiu")

        conn.close()


def main():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind(("0.0.0.0", 5000))
    server.listen()

    print("Servidor iniciado na porta 5000...")

    while True:
        conn, addr = server.accept()
        thread = threading.Thread(target=handle_client, args=(conn, addr))
        thread.start()


if __name__ == "__main__":
    main()
