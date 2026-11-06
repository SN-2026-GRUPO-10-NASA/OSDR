"""Robô coletor: busca estudos na API de busca da OSDR e grava no Supabase.

Variáveis de ambiente (vêm dos Secrets do GitHub Actions):
    SUPABASE_URL          URL do projeto Supabase
    SUPABASE_SERVICE_KEY  chave secret/service_role (nunca no código)
    OSDR_TERM             termo de busca (opcional, padrão: vazio = todos)
"""

import json
import os
import sys
from datetime import datetime, timezone

import requests

API_BUSCA = "https://osdr.nasa.gov/osdr/data/search"
TAMANHO_PAGINA = 50
TAMANHO_LOTE = 100
TIMEOUT = 30


def registrar_execucao(url, chave, status, quantidade, mensagem, iniciado):
    """Grava uma linha na tabela execucoes (diário do robô)."""
    corpo = {
        "iniciado_em": iniciado.isoformat(),
        "finalizado_em": datetime.now(timezone.utc).isoformat(),
        "status": status,
        "registros_coletados": quantidade,
        "mensagem": mensagem[:500],
    }
    requests.post(
        f"{url}/rest/v1/execucoes",
        headers=cabecalhos(chave),
        data=json.dumps(corpo),
        timeout=TIMEOUT,
    )


def cabecalhos(chave, extra=None):
    cab = {
        "apikey": chave,
        "Authorization": f"Bearer {chave}",
        "Content-Type": "application/json",
    }
    if extra:
        cab.update(extra)
    return cab


def buscar_pagina(termo, inicio):
    """Busca uma página de estudos na API da OSDR."""
    params = {
        "term": termo,
        "from": inicio,
        "size": TAMANHO_PAGINA,
        "type": "cgene",
    }
    resposta = requests.get(API_BUSCA, params=params, timeout=TIMEOUT)
    resposta.raise_for_status()
    dados = resposta.json()
    hits = dados.get("hits", {})
    total = hits.get("total", 0)
    if isinstance(total, dict):  # formato {"value": N, "relation": "eq"}
        total = total.get("value", 0)
    itens = [h.get("_source", h) for h in hits.get("hits", [])]
    return itens, total


def data_epoch(valor):
    """Converte epoch (segundos ou milissegundos) em data ISO. Retorna None se inválido."""
    try:
        numero = float(valor)
    except (TypeError, ValueError):
        return None
    if numero > 1e11:  # veio em milissegundos
        numero /= 1000
    return datetime.fromtimestamp(numero, timezone.utc).date().isoformat()


def normalizar(item):
    """Transforma um registro bruto da API nas colunas da tabela."""
    accession = item.get("Accession")
    if not accession:
        return None
    missao = item.get("Mission") or {}
    if isinstance(missao, list):
        missao = missao[0] if missao else {}
    return {
        "accession": accession,
        "titulo": item.get("Study Title"),
        "descricao": item.get("Study Description"),
        "organismo": item.get("organism"),
        "tipo_material": item.get("Material Type"),
        "tipo_projeto": item.get("Project Type"),
        "missao": missao.get("Name"),
        "missao_inicio": missao.get("Start Date"),
        "missao_fim": missao.get("End Date"),
        "tecnologia_ensaio": item.get("Study Assay Technology Type"),
        "tipo_medicao": item.get("Study Assay Measurement Type"),
        "data_publicacao": data_epoch(item.get("Study Public Release Date")),
        "identificadores": ", ".join(item.get("Identifiers") or [])
        if isinstance(item.get("Identifiers"), list)
        else item.get("Identifiers"),
        "url": f"https://osdr.nasa.gov/bio/repo/data/studies/{accession}",
        "atualizado_em": datetime.now(timezone.utc).isoformat(),
    }


def gravar_lote(url, chave, lote):
    """Upsert em lote: atualiza se a accession já existe, insere se não."""
    resposta = requests.post(
        f"{url}/rest/v1/osdr_estudos?on_conflict=accession",
        headers=cabecalhos(chave, {"Prefer": "resolution=merge-duplicates"}),
        data=json.dumps(lote),
        timeout=TIMEOUT,
    )
    resposta.raise_for_status()


def main():
    url = os.environ["SUPABASE_URL"].rstrip("/")
    chave = os.environ["SUPABASE_SERVICE_KEY"]
    termo = os.environ.get("OSDR_TERM", "")
    iniciado = datetime.now(timezone.utc)

    estudos = {}  # dicionário evita duplicados pela accession
    erros = []
    try:
        inicio = 0
        while True:
            itens, total = buscar_pagina(termo, inicio)
            for item in itens:
                normal = normalizar(item)
                if normal:
                    estudos[normal["accession"]] = normal
            inicio += TAMANHO_PAGINA
            if not itens or inicio >= total:
                break
        print(f"Coletados {len(estudos)} estudos únicos.")
    except Exception as exc:  # falha na API: nada foi gravado ainda
        mensagem = f"Falha ao buscar na API: {exc}"
        print(mensagem, file=sys.stderr)
        registrar_execucao(url, chave, "erro_critico", 0, mensagem, iniciado)
        sys.exit(1)

    lista = list(estudos.values())
    for i in range(0, len(lista), TAMANHO_LOTE):
        lote = lista[i : i + TAMANHO_LOTE]
        try:
            gravar_lote(url, chave, lote)
        except Exception as exc:
            erros.append(f"lote {i // TAMANHO_LOTE}: {exc}")
            print(erros[-1], file=sys.stderr)

    if erros:
        status = "erro_parcial"
        mensagem = "; ".join(erros)
    else:
        status = "concluido"
        mensagem = f"{len(lista)} estudos gravados"

    registrar_execucao(url, chave, status, len(lista), mensagem, iniciado)
    print(mensagem)
    sys.exit(1 if erros else 0)


if __name__ == "__main__":
    main()
