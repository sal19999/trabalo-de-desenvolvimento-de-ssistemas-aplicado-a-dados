# trabalo-de-desenvolvimento-de-ssistemas-aplicado-a-dados
import socket
import threading
import time

import httpx
import requests
import uvicorn
from fastapi import FastAPI, HTTPException
from gtts import gTTS
from IPython.display import Audio

# ---------------------------------------------------------------
# Fontes de cotação (se uma falhar, tenta a próxima)
# ---------------------------------------------------------------
def fonte_awesomeapi():
    r = httpx.get("https://economia.awesomeapi.com.br/json/last/USD-BRL", timeout=10)
    r.raise_for_status()
    return float(r.json()["USDBRL"]["bid"])


def fonte_open_er_api():
    r = httpx.get("https://open.er-api.com/v6/latest/USD", timeout=10)
    r.raise_for_status()
    return float(r.json()["rates"]["BRL"])


FONTES = [("AwesomeAPI", fonte_awesomeapi), ("ExchangeRate-API", fonte_open_er_api)]


def formatar_reais(valor: float) -> str:
    return f"R$ {valor:,.2f}".replace(",", "X").replace(".", ",").replace("X", ".")


# ---------------------------------------------------------------
# API REST
# ---------------------------------------------------------------
app = FastAPI(title="API de Cotação do Dólar")


@app.get("/")
def inicio():
    return {"mensagem": "API no ar. Acesse /cotacao para ver o valor do dólar."}


@app.get("/cotacao")
def cotacao():
    erros = []
    for nome, buscar in FONTES:
        try:
            valor = buscar()
            return {
                "moeda": "USD/BRL",
                "cotacao": round(valor, 4),
                "fonte": nome,
                "mensagem": f"Hoje, 1 dólar está valendo {formatar_reais(valor)}.",
            }
        except Exception as e:
            erros.append(f"{nome}: {e}")
    raise HTTPException(status_code=502, detail={"erro": "Nenhuma fonte respondeu.", "detalhes": erros})


# ---------------------------------------------------------------
# Sobe o servidor em segundo plano (necessário no Colab)
# ---------------------------------------------------------------
def porta_livre() -> int:
    with socket.socket() as s:
        s.bind(("127.0.0.1", 0))
        return s.getsockname()[1]


PORTA = porta_livre()
server = uvicorn.Server(uvicorn.Config(app, host="127.0.0.1", port=PORTA, log_level="warning"))
threading.Thread(target=server.run, daemon=True).start()

for _ in range(40):  # espera até 20 segundos
    if server.started:
        break
    time.sleep(0.5)
else:
    raise RuntimeError("O servidor não iniciou. Reinicie a sessão e rode de novo.")

print(f"Servidor rodando na porta {PORTA}")

# ---------------------------------------------------------------
# Cliente: consulta a API e fala o resultado para o usuário
# ---------------------------------------------------------------
resposta = requests.get(f"http://127.0.0.1:{PORTA}/cotacao", timeout=30)

if resposta.status_code == 200:
    dados = resposta.json()
    print(dados["mensagem"])
    print(f"(fonte: {dados['fonte']})")

    gTTS(dados["mensagem"], lang="pt").save("cotacao.mp3")
    display(Audio("cotacao.mp3", autoplay=True))
else:
    print("Erro ao consultar a cotação:", resposta.status_code)
    print(resposta.json())
