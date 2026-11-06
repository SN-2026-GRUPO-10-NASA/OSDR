# OSDR: Pipeline de dados da NASA (Grupo 10)

Painel que coleta estudos de biologia espacial do **NASA Open Science Data Repository (OSDR)** e os exibe em uma página web.

## Arquitetura

1. **API da OSDR:** fonte dos dados (`https://osdr.nasa.gov/osdr/data/search`, sem chave).
2. **GitHub Actions:** roda `scripts/fetch_nasa.py` todos os dias às 06:00 UTC (também pode ser executado manualmente).
3. **Supabase:** banco de dados PostgreSQL (tabelas `osdr_estudos` e `execucoes`).
4. **GitHub Pages:** `index.html`, o painel que lê o Supabase pela API REST.

## Como configurar

1. Criar o projeto no Supabase (região South America, São Paulo) e rodar `sql/setup.sql` no SQL Editor.
2. Em Settings → Secrets and variables → Actions, criar:
   - `SUPABASE_URL`
   - `SUPABASE_SERVICE_KEY` (chave secret/service_role; nunca em código)
3. Em Actions, executar **Atualizar dados OSDR** (Run workflow) e conferir o ícone verde.
4. No `index.html`, trocar `COLE_AQUI_A_URL_DO_PROJETO` e `COLE_AQUI_A_PUBLISHABLE_KEY` pela URL e pela publishable/anon key.
5. Em Settings → Pages, ativar a publicação a partir de `main` / `/ (root)`.

## iNTEGRANTES

| Integrante |
|---|---|
| (Julia) | (Eduarda) (Ana) |

## Links do Projeto

Link do SupaBase: https://gbwgwmaxzcqchbiwcpvj.supabase.co
Link do Pages: https://sn-2026-grupo-10-nasa.github.io/OSDR/
Link da API: https://osdr.nasa.gov/bio/repo/search?q=&data_source=cgene,alsda,esa&data_type=study



