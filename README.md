# CLACS GeoModel — Backend

Backend multiempresa (SaaS) para gestão de dados de geologia mineira. Cobre o MVP com **20 tabelas**,
isolamento por empresa, permissões por perfil, auditoria, validações geológicas, importação de CSV/XLSX,
dashboard e estimativa de recursos.

- **Stack:** FastAPI, SQLAlchemy 2.0, Alembic, Pydantic v2, SQLite (MVP), JWT + bcrypt.
- **Localização:** `C:\Users\Suporte\Desktop\clacs-geomodel`
- **Documentação interactiva:** `http://127.0.0.1:8000/docs`

---

## 1. Arranque rápido (Windows PowerShell)

```powershell
cd C:\Users\Suporte\Desktop\clacs-geomodel

# O .venv já existe neste ambiente. Se precisar de recriar:
# C:\Users\Suporte\AppData\Local\Programs\Python\Python313\python.exe -m venv --system-site-packages .venv

.\.venv\Scripts\Activate.ps1

# Base de dados + migração
.\.venv\Scripts\python.exe -m alembic upgrade head

# Permissões, super administrador e empresa demo
.\.venv\Scripts\python.exe scripts\seed.py

# API
.\.venv\Scripts\python.exe -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Abrir `http://127.0.0.1:8000/docs`.

> O ficheiro `.env` já está criado para desenvolvimento. Em produção altere `JWT_SECRET`,
> `ADMIN_PASSWORD`, `CORS_ORIGINS` e defina `AUTO_CRIAR_TABELAS=false`.

### Sem dados demo

```powershell
.\.venv\Scripts\python.exe scripts\seed.py --sem-demo
```

---

## 2. Credenciais semeadas

| Perfil | E-mail | Senha | Empresa |
|---|---|---|---|
| Super administrador CLACS | `admin@clacs.ao` | `admin123` | todas |
| Administrador da empresa demo | `gestor@clacs.ao` | `Demo12345` | `clacs-demo` |
| Geólogo | `geologo@clacs.ao` | `Demo12345` | `clacs-demo` |
| Técnico de laboratório | `laboratorio@clacs.ao` | `Demo12345` | `clacs-demo` |
| Engenheiro de minas | `engenheiro@clacs.ao` | `Demo12345` | `clacs-demo` |
| Auditor | `auditor@clacs.ao` | `Demo12345` | `clacs-demo` |

**Altere estas senhas antes de qualquer utilização real.**

---

## 3. Autenticação e empresa activa

1. `POST /auth/login` com `{ "email": "...", "senha": "..." }`.
2. Usar o token: `Authorization: Bearer <access_token>`.
3. Seleccionar a empresa com o cabeçalho `X-Company-ID: <uuid>`.
   - Utilizadores com uma única empresa podem omitir o cabeçalho.
   - Utilizadores com várias empresas são obrigados a indicá-lo.
   - O super administrador pode actuar sem empresa para gerir clientes.

Exemplo:

```powershell
$login = Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8000/auth/login `
  -ContentType 'application/json' `
  -Body '{"email":"gestor@clacs.ao","senha":"Demo12345"}'

$headers = @{
  Authorization = "Bearer $($login.access_token)"
  "X-Company-ID" = $login.empresa_actual.id
}

Invoke-RestMethod -Uri http://127.0.0.1:8000/dashboard -Headers $headers
```

---

## 4. Modelo de dados (20 tabelas)

| # | Tabela | Função |
|---|---|---|
| 1 | `companies` | Empresas/clientes SaaS |
| 2 | `users` | Utilizadores globais |
| 3 | `company_users` | Associação utilizador ↔ empresa ↔ perfil |
| 4 | `roles` | Perfis por empresa |
| 5 | `permissions` | Catálogo de permissões |
| 6 | `role_permissions` | Permissões atribuídas a cada perfil |
| 7 | `projects` | Projectos mineiros |
| 8 | `project_areas` | Áreas/sub-áreas do projecto |
| 9 | `lithologies` | Catálogo de litologias |
| 10 | `drill_holes` | Furos de sondagem (collar) |
| 11 | `drill_surveys` | Medições de trajectória |
| 12 | `drill_intervals` | Intervalos geológicos |
| 13 | `samples` | Amostras |
| 14 | `laboratories` | Laboratórios |
| 15 | `assay_results` | Resultados laboratoriais |
| 16 | `geological_models` | Modelos geológicos 3D |
| 17 | `geological_domains` | Domínios geológicos |
| 18 | `model_versions` | Versões e metadados de ficheiros 3D |
| 19 | `resource_estimates` | Estimativas de recursos |
| 20 | `audit_logs` | Auditoria de operações |

Todas as entidades de negócio têm `company_id`, chaves estrangeiras explícitas, índices e UUID em texto.
`projects`, `drill_holes`, `samples` e `geological_models` usam soft delete (`eliminado_em`).

---

## 5. Perfis e permissões

Módulos: `companies`, `users`, `projects`, `drill_holes`, `samples`, `laboratories`, `assays`,
`models`, `resources`, `dashboard`, `audit`, `imports`.

Acções: `view`, `create`, `edit`, `delete`, `validate`, `import`, `export`.

Perfis de sistema criados por empresa:

- `super_admin` (CLACS, acesso total)
- `company_admin` (acesso total à empresa)
- `project_manager`
- `geologist`
- `drill_technician`
- `lab_technician`
- `mining_engineer`
- `resource_analyst`
- `auditor`
- `viewer`

A matriz pode ser consultada em `GET /roles` e alterada em `PATCH /roles/{id}`.
O catálogo completo está em `GET /permissions`.

---

## 6. Endpoints principais

### Autenticação e administração

| Método | Rota | Descrição |
|---|---|---|
| POST | `/auth/login` | Iniciar sessão |
| GET | `/auth/me` | Perfil, empresa, papel e permissões |
| POST | `/auth/alterar-senha` | Alterar a própria senha |
| GET/POST | `/companies` | Listar/criar empresas (criar: super admin) |
| GET/PATCH/DELETE | `/companies/{id}` | Consultar/actualizar/desactivar |
| GET/POST | `/users` | Membros da empresa / criar ou associar |
| PATCH/DELETE | `/users/{id}` | Actualizar/desactivar membro |
| PATCH | `/users/{id}/membro` | Alterar perfil ou estado |
| GET/POST | `/roles` | Perfis da empresa |
| PATCH/DELETE | `/roles/{id}` | Actualizar/apagar perfil personalizado |
| GET | `/permissions` | Catálogo de permissões |

### Projectos e litologias

| Método | Rota |
|---|---|
| GET/POST | `/projects` |
| GET/PATCH/DELETE | `/projects/{id}` |
| GET/POST | `/projects/{id}/areas` |
| PATCH/DELETE | `/projects/{id}/areas/{area_id}` |
| GET/POST | `/lithologies` |
| PATCH/DELETE | `/lithologies/{id}` |

### Sondagens

| Método | Rota |
|---|---|
| GET/POST | `/drill-holes` |
| GET/PATCH/DELETE | `/drill-holes/{id}` |
| GET/POST | `/drill-holes/{id}/surveys` |
| PATCH/DELETE | `/drill-holes/{id}/surveys/{survey_id}` |
| GET | `/drill-holes/{id}/trajectory` |
| GET/POST | `/drill-holes/{id}/intervals` |
| PATCH/DELETE | `/drill-holes/{id}/intervals/{interval_id}` |

### Amostras, laboratórios e resultados

| Método | Rota |
|---|---|
| GET/POST | `/samples` |
| GET/PATCH/DELETE | `/samples/{id}` |
| GET/POST | `/laboratories` |
| GET/PATCH/DELETE | `/laboratories/{id}` |
| GET/POST | `/assays` |
| GET/PATCH/DELETE | `/assays/{id}` |
| POST | `/assays/{id}/validate` |

### Modelação 3D e recursos

| Método | Rota |
|---|---|
| GET/POST | `/models` |
| GET/PATCH/DELETE | `/models/{id}` |
| GET/POST | `/models/{id}/domains` |
| PATCH/DELETE | `/models/{id}/domains/{domain_id}` |
| GET/POST | `/models/{id}/versions` |
| POST | `/models/{id}/versions/{version_id}/validate` |
| DELETE | `/models/{id}/versions/{version_id}` |
| GET/POST | `/resources` |
| GET/PATCH/DELETE | `/resources/{id}` |
| POST | `/resources/{id}/validate` |

### Dashboard, auditoria e importação

| Método | Rota | Descrição |
|---|---|---|
| GET | `/dashboard` | Indicadores calculados dos módulos |
| GET | `/audit` | Consulta com filtros |
| GET | `/audit/export` | Exportação CSV |
| GET | `/imports/entities` | Entidades, campos e aliases |
| POST | `/imports/preview/{entidade}` | Pré-visualizar sem gravar |
| POST | `/imports/{entidade}` | Validar e gravar (atómico) |

---

## 7. Validações implementadas

- **Profundidades:** não negativas; `profundidade_final ≤ profundidade_prevista`; não inferior a surveys/intervalos existentes.
- **Surveys:** profundidade única por furo e dentro da profundidade final.
- **Intervalos:** `ate > de`; sem sobreposição; dentro da profundidade final; litologia da mesma empresa.
- **Amostras:** código único por empresa; furo e intervalo da mesma empresa/projecto; profundidade dentro do furo e do intervalo.
- **Resultados:** amostra e laboratório da empresa; laboratório activo; sem duplicado amostra/laboratório/parâmetro/método; validação exige permissão.
- **Modelos:** nome único por projecto; domínio único por modelo; versões numeradas e validação com permissão.
- **Recursos:** projecto/modelo coerentes; `tonelagem = volume × densidade`; metal contido convertido de `%`, `ppm`, `g/t`, `ppb` ou fracção.
- **Datas:** `data_fim ≥ data_inicio` em projectos e furos.
- **GeoJSON:** `type` válido e `coordinates` quando aplicável.
- **Multiempresa:** todas as consultas e escritas filtram `company_id`; acessos cruzados devolvem `404`/`422`/`403`.

---

## 8. Importação CSV/XLSX

Entidades aceites: `sondagens` (`drill_holes`), `amostras` (`samples`), `resultados` (`assays`) e
`litologias` (`lithologies`).

- Aceita `;`, `,` ou tabulação em CSV; `.xlsx`/`.xlsm` via primeira folha.
- Cabeçalhos sem acentos, maiúsculas/minúsculas ou com aliases (ex.: `projecto`, `projeto`, `project`).
- Números com vírgula decimal (`1,2`) e datas `AAAA-MM-DD` ou `DD/MM/AAAA`.
- **Pré-visualização** não grava nada.
- **Confirmação** é atómica: se houver um erro, nada é gravado.
- Erros são devolvidos por linha, com número e mensagem.

Exemplo de CSV de sondagens:

```csv
projecto;codigo;profundidade_prevista;profundidade_final;x;y;z;azimute;inclinacao
PRJ-001;FUR-100;250;180;10;20;1200;90;-60
PRJ-001;FUR-101;300;220;35;42;1185;95;-65
```

Exemplo de CSV de resultados:

```csv
amostra;laboratorio;parametro;valor;unidade;metodo
AM-001;ALS;Cu;1.2;%;ICP-OES
AM-002;ALS;Au;0.5;ppm;Fire Assay
```

Ver aliases completos em `GET /imports/entities`.

---

## 9. Testes

```powershell
.\.venv\Scripts\python.exe -m pytest -q
```

Cobertura actual: **21 testes** (autenticação, isolamento entre empresas, permissões, sondagens,
trajectória, amostras, resultados, importação CSV/XLSX, modelos/versões, estimativas, dashboard,
auditoria e administração).

Os testes usam `test_geomodel.db`, recriada automaticamente e independente de `geomodel.db`.

---

## 10. Migrações

```powershell
# Aplicar
.\.venv\Scripts\python.exe -m alembic upgrade head

# Criar nova migração após alterar modelos
.\.venv\Scripts\python.exe -m alembic revision --autogenerate -m "descricao"

# Reverter uma migração
.\.venv\Scripts\python.exe -m alembic downgrade -1
```

A migração inicial está em `alembic/versions/`.

---

## 11. Estrutura

```text
clacs-geomodel/
├── app/
│   ├── main.py              # aplicação FastAPI
│   ├── config.py            # configuração/.env
│   ├── database.py          # engine, sessão, Base
│   ├── models.py            # 20 tabelas
│   ├── schemas.py           # Pydantic v2
│   ├── security.py          # JWT + bcrypt
│   ├── deps.py              # autenticação, empresa, permissões
│   ├── common.py            # paginação e normalização
│   ├── routers/             # auth, companies, users, projects, drill_holes,
│   │                        # samples, laboratories, assays, models3d,
│   │                        # resources, dashboard, audit, importacao
│   └── services/            # auditoria, geo, perfis, importacao
├── alembic/                 # migrações
├── scripts/seed.py          # seed + demo
├── tests/                   # pytest
├── requirements.txt
├── alembic.ini
├── .env.example
├── frontend/                # React 19 + Vite 7 + TypeScript (UI)
└── README.md
```

---

## 12. Frontend (React + Vite)

A interface web está em `frontend/` e cobre todos os módulos do backend: dashboard, projectos,
sondagens, amostras, laboratórios, resultados, modelação 3D, recursos, importação, utilizadores,
perfis, auditoria, empresas e definições da conta.

### Arranque em desenvolvimento

```powershell
cd frontend
npm install
npm run dev
```

A aplicação fica em `http://localhost:5174`. O Vite encaminha `/api` para `http://127.0.0.1:8000`
e remove o prefixo, pelo que o FastAPI deve estar a correr em paralelo:

```powershell
.\.venv\Scripts\python.exe -m uvicorn app.main:app --reload
```

### Build de produção

```powershell
cd frontend
npm run build
npm run preview
```

O resultado é gerado em `frontend/dist`. Para apontar para outra API, definir `VITE_API_URL`
(por exemplo `https://api.clacs.ao`). A autenticação usa JWT e a empresa activa é enviada no
cabeçalho `X-Company-ID`.

### Stack e organização

- React 19, TypeScript estrito, React Router 7, Vite 7 e `lucide-react`.
- Sistema de design próprio em `frontend/src/styles/global.css` (tema claro/escuro, layout
  responsivo, componentes de tabela, formulários, gráficos SVG e mapa de projectos).
- `src/app/services/` contém o cliente HTTP, autenticação multi-empresa, tipos e formatação.
- `src/app/pages/` contém uma página por módulo; `src/app/App.tsx` define as rotas protegidas
  por permissões.

---

## 13. Limitações do MVP e próximos passos

- **SQLite:** não suporta PostGIS. As geometrias são GeoJSON em colunas JSON. Para PostgreSQL/PostGIS,
  trocar `DATABASE_URL`, instalar `psycopg` e migrar as colunas de geometria.
- **Ficheiros 3D:** o backend guarda metadados, chave e URL; os ficheiros pesados (OBJ/STL/LAS/DXF/SHP/
  GeoJSON/PDF) devem ficar em object storage (S3/MinIO).
- **Importação:** CSV/XLSX já implementados. DXF/SHP/GeoJSON/LAS/OBJ/STL/PDF ficam para fase seguinte.
- **Autenticação:** JWT próprio. Keycloak/SSO fica para produção.
- **Cálculo de recursos:** MVP por volume × densidade e teor médio. Kriging/IDW real e classificação de
  recursos (medidos/indicados/inferidos) ficam para fase seguinte.
- **Frontend:** implementado em `frontend/` com React + Vite. A UI consome os endpoints autenticados
  descritos acima; a documentação interactiva continua disponível em `/docs`.

---

## 14. Segurança antes de produção

1. Alterar `JWT_SECRET` para um valor aleatório longo.
2. Alterar `ADMIN_PASSWORD` e todas as senhas demo.
3. Restringir `CORS_ORIGINS` aos domínios reais.
4. Usar HTTPS e um reverse proxy.
5. Definir `AUTO_CRIAR_TABELAS=false` e aplicar `alembic upgrade head`.
6. Fazer backups regulares da base de dados e do object storage.
7. Rotacionar credenciais que tenham sido partilhadas em texto simples.
