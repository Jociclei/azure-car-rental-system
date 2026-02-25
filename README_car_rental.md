# 🚗 Sistema de Aluguel de Carros na Nuvem com Microsoft Azure
> Projeto desenvolvido como parte do desafio prático da DIO — Microsoft Application Platform

---

## 📋 Descrição do Projeto

Solução completa em nuvem para **gerenciamento de sistema de aluguel de carros**, cobrindo desde o back-end da API até o ambiente de hospedagem na **Microsoft Azure**. O sistema gerencia frota de veículos, reservas, clientes, contratos e devoluções com foco em escalabilidade e disponibilidade.

---

## 🏗️ Arquitetura da Solução

```
┌────────────────────────────────────────────────────────────────┐
│                    CLIENTES                                    │
│      App Mobile • Web App • Quiosques nas Filiais              │
└──────────────────────────┬─────────────────────────────────────┘
                           │ HTTPS
              ┌────────────▼────────────┐
              │   Azure Front Door      │
              │  CDN + WAF + SSL        │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │  Azure API Management   │
              │  Auth • Rate Limit      │
              │  Versionamento de API   │
              └────────────┬────────────┘
                           │
     ┌─────────────────────┼──────────────────────┐
     │                     │                      │
┌────▼─────┐         ┌─────▼──────┐        ┌──────▼─────┐
│  Veículos │         │  Reservas  │        │  Clientes  │
│   API     │         │    API     │        │    API     │
│(App Svc)  │         │ (App Svc)  │        │ (App Svc)  │
└────┬──────┘         └─────┬──────┘        └──────┬─────┘
     │                      │                      │
     └──────────────┬────────┘──────────────────────┘
                    │
┌───────────────────▼──────────────────────────────────┐
│                  DADOS & STORAGE                      │
│  ┌─────────────┐ ┌─────────────┐ ┌────────────────┐  │
│  │  Azure SQL  │ │   Cosmos DB │ │  Blob Storage  │  │
│  │  (Reservas, │ │  (Histórico │ │  (Fotos dos    │  │
│  │  Contratos) │ │  de uso)    │ │   veículos)    │  │
│  └─────────────┘ └─────────────┘ └────────────────┘  │
│  ┌─────────────┐ ┌─────────────┐                      │
│  │   Redis     │ │  Key Vault  │                      │
│  │   Cache     │ │  (Secrets)  │                      │
│  └─────────────┘ └─────────────┘                      │
└──────────────────────────────────────────────────────┘
                    │
┌───────────────────▼──────────────────────────────────┐
│             EVENTOS & NOTIFICAÇÕES                    │
│   Azure Service Bus • Azure Functions                 │
│   (Confirmação reserva • Lembrete devolução)          │
└──────────────────────────────────────────────────────┘
```

---

## ☁️ Serviços Azure Utilizados

| Serviço | Finalidade |
|---|---|
| **Azure App Service** | Hospedagem das APIs (Veículos, Reservas, Clientes) |
| **Azure SQL Database** | Reservas, contratos, pagamentos e frota |
| **Azure Cosmos DB** | Histórico de uso e telemetria de veículos |
| **Azure Blob Storage** | Fotos dos veículos, CNH digitalizada, contratos PDF |
| **Azure Cache for Redis** | Disponibilidade de veículos em tempo real |
| **Azure API Management** | Gateway, autenticação e versionamento |
| **Azure AD B2C** | Autenticação de clientes e funcionários |
| **Azure Key Vault** | Secrets e certificados |
| **Azure Service Bus** | Notificações assíncronas de reserva/devolução |
| **Azure Functions** | Lembretes automáticos e cálculo de multas |
| **Azure Maps** | Localização das filiais e rastreamento de frota |
| **Application Insights** | Monitoramento e telemetria |

---

## 🗄️ Modelagem de Dados

### Azure SQL — Schema Principal

```sql
-- Frota de veículos
CREATE TABLE Veiculos (
    id               UNIQUEIDENTIFIER  PRIMARY KEY DEFAULT NEWID(),
    placa            CHAR(7)           NOT NULL UNIQUE,
    marca            NVARCHAR(50)      NOT NULL,
    modelo           NVARCHAR(100)     NOT NULL,
    ano              SMALLINT          NOT NULL,
    categoria        NVARCHAR(30)      NOT NULL, -- economico|intermediario|suv|premium|van
    cor              NVARCHAR(30)      NOT NULL,
    quilometragem    INT               NOT NULL DEFAULT 0,
    status           NVARCHAR(20)      NOT NULL DEFAULT 'disponivel',
    -- disponivel | alugado | manutencao | reservado
    valor_diaria     DECIMAL(8,2)      NOT NULL,
    filial_id        UNIQUEIDENTIFIER  REFERENCES Filiais(id),
    foto_principal   NVARCHAR(500),    -- URL do Blob Storage
    created_at       DATETIME2         DEFAULT GETDATE()
);

-- Clientes
CREATE TABLE Clientes (
    id               UNIQUEIDENTIFIER  PRIMARY KEY DEFAULT NEWID(),
    nome             NVARCHAR(150)     NOT NULL,
    cpf              CHAR(11)          NOT NULL UNIQUE,
    email            NVARCHAR(200)     NOT NULL UNIQUE,
    telefone         NVARCHAR(20),
    cnh_numero       NVARCHAR(20)      NOT NULL,
    cnh_validade     DATE              NOT NULL,
    cnh_url          NVARCHAR(500),    -- Foto da CNH no Blob Storage
    bloqueado        BIT               DEFAULT 0,
    created_at       DATETIME2         DEFAULT GETDATE()
);

-- Reservas e Contratos
CREATE TABLE Reservas (
    id               UNIQUEIDENTIFIER  PRIMARY KEY DEFAULT NEWID(),
    cliente_id       UNIQUEIDENTIFIER  REFERENCES Clientes(id),
    veiculo_id       UNIQUEIDENTIFIER  REFERENCES Veiculos(id),
    filial_retirada  UNIQUEIDENTIFIER  REFERENCES Filiais(id),
    filial_devolucao UNIQUEIDENTIFIER  REFERENCES Filiais(id),
    data_retirada    DATETIME2         NOT NULL,
    data_devolucao   DATETIME2         NOT NULL,
    data_devolucao_real DATETIME2,
    dias_previstos   AS DATEDIFF(DAY, data_retirada, data_devolucao),
    valor_diaria     DECIMAL(8,2)      NOT NULL,
    valor_total      DECIMAL(10,2),
    valor_multa      DECIMAL(10,2)     DEFAULT 0,
    status           NVARCHAR(20)      NOT NULL DEFAULT 'confirmada',
    -- confirmada | ativa | concluida | cancelada
    km_saida         INT,
    km_chegada       INT,
    contrato_url     NVARCHAR(500),    -- PDF no Blob Storage
    created_at       DATETIME2         DEFAULT GETDATE()
);

-- Filiais
CREATE TABLE Filiais (
    id               UNIQUEIDENTIFIER  PRIMARY KEY DEFAULT NEWID(),
    nome             NVARCHAR(150)     NOT NULL,
    cidade           NVARCHAR(100)     NOT NULL,
    estado           CHAR(2)           NOT NULL,
    latitude         DECIMAL(10,7),
    longitude        DECIMAL(10,7),
    telefone         NVARCHAR(20),
    ativa            BIT               DEFAULT 1
);

-- Índices
CREATE INDEX idx_veiculos_status    ON Veiculos(status, categoria, filial_id);
CREATE INDEX idx_reservas_datas     ON Reservas(data_retirada, data_devolucao, status);
CREATE INDEX idx_reservas_cliente   ON Reservas(cliente_id, status);
```

---

## 🚀 Passo a Passo da Implementação

### 1. Criar Resource Group e Identidade
```bash
az group create \
  --name rg-car-rental-dio \
  --location brazilsouth

az identity create \
  --name id-car-rental-api \
  --resource-group rg-car-rental-dio
```

### 2. Azure SQL Database
```bash
az sql server create \
  --name sql-car-rental-dio \
  --resource-group rg-car-rental-dio \
  --location brazilsouth \
  --admin-user rentaladmin \
  --admin-password "CarRental@2024!"

az sql db create \
  --resource-group rg-car-rental-dio \
  --server sql-car-rental-dio \
  --name db-rental \
  --service-objective S2
```

### 3. Azure Cosmos DB — Histórico de Telemetria
```bash
az cosmosdb create \
  --name cosmos-car-rental-dio \
  --resource-group rg-car-rental-dio \
  --kind GlobalDocumentDB \
  --locations regionName=brazilsouth

az cosmosdb sql database create \
  --account-name cosmos-car-rental-dio \
  --resource-group rg-car-rental-dio \
  --name TelemetriaDB

az cosmosdb sql container create \
  --account-name cosmos-car-rental-dio \
  --resource-group rg-car-rental-dio \
  --database-name TelemetriaDB \
  --name historicoUso \
  --partition-key-path "/veiculo_id"
```

### 4. Azure Blob Storage — Mídias e Documentos
```bash
az storage account create \
  --name stcarrentaldio \
  --resource-group rg-car-rental-dio \
  --location brazilsouth \
  --sku Standard_LRS

# Containers para cada tipo de arquivo
for container in fotos-veiculos documentos-clientes contratos-pdf laudos-vistoria; do
  az storage container create \
    --account-name stcarrentaldio \
    --name $container \
    --public-access off
done
```

### 5. Azure Cache for Redis — Disponibilidade em Tempo Real
```bash
az redis create \
  --name redis-car-rental-dio \
  --resource-group rg-car-rental-dio \
  --location brazilsouth \
  --sku Standard \
  --vm-size c1
```

```python
# Estratégia de cache para disponibilidade de veículos
# Atualizado a cada reserva/devolução — TTL de 2 minutos

def buscar_veiculos_disponiveis(categoria: str, filial_id: str, 
                                 data_retirada: str, data_devolucao: str):
    cache_key = f"disponivel:{filial_id}:{categoria}:{data_retirada}:{data_devolucao}"
    
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    veiculos = db.query("""
        SELECT v.* FROM Veiculos v
        WHERE v.filial_id = ? AND v.categoria = ? AND v.status = 'disponivel'
        AND v.id NOT IN (
            SELECT veiculo_id FROM Reservas
            WHERE status IN ('confirmada', 'ativa')
            AND data_retirada < ? AND data_devolucao > ?
        )
    """, filial_id, categoria, data_devolucao, data_retirada)
    
    redis.setex(cache_key, 120, json.dumps(veiculos))
    return veiculos
```

### 6. Azure App Service — Deploy das APIs
```bash
az appservice plan create \
  --name plan-car-rental-dio \
  --resource-group rg-car-rental-dio \
  --sku B2 --is-linux

for api in veiculos reservas clientes; do
  az webapp create \
    --resource-group rg-car-rental-dio \
    --plan plan-car-rental-dio \
    --name app-$api-dio \
    --runtime "NODE:20-lts"
done
```

### 7. Azure Function — Cálculo de Multa por Atraso
```python
import azure.functions as func
from datetime import datetime

app = func.FunctionApp()

@app.timer_trigger(schedule="0 */30 * * * *")  # A cada 30 minutos
def verificar_devolucoes_atrasadas(timer: func.TimerRequest):
    """Verifica reservas com devolução atrasada e aplica multa automaticamente"""
    reservas_atrasadas = db.query("""
        SELECT * FROM Reservas
        WHERE status = 'ativa'
        AND data_devolucao < GETDATE()
        AND data_devolucao_real IS NULL
    """)
    
    for reserva in reservas_atrasadas:
        dias_atraso = (datetime.now() - reserva.data_devolucao).days
        valor_multa = dias_atraso * reserva.valor_diaria * 1.5  # 50% de acréscimo
        
        db.execute("""
            UPDATE Reservas SET valor_multa = ? WHERE id = ?
        """, valor_multa, reserva.id)
        
        # Notificar cliente via Service Bus
        service_bus.send({
            "tipo": "notificacao_atraso",
            "cliente_id": reserva.cliente_id,
            "reserva_id": str(reserva.id),
            "dias_atraso": dias_atraso,
            "valor_multa": valor_multa
        })
```

---

## 📡 Endpoints da API

```
# Veículos
GET    /api/v1/veiculos                    → Listar veículos disponíveis com filtros
GET    /api/v1/veiculos/{id}               → Detalhes do veículo
GET    /api/v1/veiculos/disponibilidade    → Checar disponibilidade por período

# Reservas
POST   /api/v1/reservas                    → Criar nova reserva
GET    /api/v1/reservas/{id}               → Detalhes da reserva
PUT    /api/v1/reservas/{id}/retirada      → Registrar retirada do veículo
PUT    /api/v1/reservas/{id}/devolucao     → Registrar devolução
DELETE /api/v1/reservas/{id}               → Cancelar reserva

# Clientes
POST   /api/v1/clientes                    → Cadastrar cliente
GET    /api/v1/clientes/{id}/historico     → Histórico de aluguéis
POST   /api/v1/clientes/{id}/documentos    → Upload de CNH

# Filiais
GET    /api/v1/filiais                     → Listar filiais com localização
GET    /api/v1/filiais/{id}/frota          → Frota disponível na filial
```

**Exemplo — Criar Reserva:**
```json
POST /api/v1/reservas
Authorization: Bearer {jwt}

{
  "veiculo_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "filial_retirada_id": "aab85f64-1234-4562-b3fc-abc3f66afa6",
  "filial_devolucao_id": "aab85f64-1234-4562-b3fc-abc3f66afa6",
  "data_retirada": "2024-12-20T09:00:00",
  "data_devolucao": "2024-12-27T09:00:00"
}
```

**Response:**
```json
{
  "id": "9fa85f64-9999-4562-b3fc-2c963f66afa9",
  "status": "confirmada",
  "veiculo": {
    "modelo": "Toyota Corolla 2024",
    "placa": "ABC1D23",
    "categoria": "intermediario"
  },
  "periodo": {
    "retirada": "2024-12-20T09:00:00",
    "devolucao": "2024-12-27T09:00:00",
    "dias": 7
  },
  "valor_total": 1050.00,
  "contrato_url": "https://stcarrentaldio.blob.core.windows.net/contratos-pdf/9fa85f64.pdf"
}
```

---

## 💡 Insights e Aprendizados

### Cosmos DB para Telemetria de Frota
Dados de GPS, quilometragem e histórico de uso de veículos têm schema variável e volume alto. O Cosmos DB com partição por `veiculo_id` foi a escolha certa — queries por veículo são extremamente rápidas e o TTL automático descarta dados antigos sem custo de manutenção.

### Redis para Disponibilidade em Tempo Real
A disponibilidade de veículos é consultada centenas de vezes por minuto durante horários de pico. Cachear por 2 minutos com invalidação imediata ao confirmar/cancelar reserva equilibrou perfeitamente performance e precisão. O Redis também armazena sessões de usuário para agilizar o checkout.

### Colunas Computadas no SQL
Usar `dias_previstos AS DATEDIFF(DAY, data_retirada, data_devolucao)` no SQL elimina lógica de negócio duplicada no código da aplicação. O banco garante consistência e o dado está sempre disponível sem cálculo extra.

### Azure Maps para UX das Filiais
Integrar Azure Maps permitiu mostrar as filiais mais próximas do usuário com tempo de deslocamento estimado. A API de routing do Azure Maps também ajudou a calcular se era mais vantajoso devolver em outra filial com taxa de conveniência.

---

## 🔮 Possibilidades de Evolução

- **Azure IoT Hub** — telemetria em tempo real dos veículos (localização GPS, nível de combustível, hodômetro)
- **Azure Machine Learning** — previsão de demanda por categoria e filial para otimizar distribuição da frota
- **Azure Computer Vision** — análise automática de fotos de vistoria para detectar avarias usando IA
- **Azure Communication Services** — envio de contratos por WhatsApp e notificações por SMS
- **Power BI Embedded** — dashboard executivo com ocupação da frota, receita por filial e inadimplência

---

## 💰 Estimativa de Custos

| Serviço | Tier | Custo/mês |
|---|---|---|
| App Service (x3 APIs) | B2 | ~$225 USD |
| Azure SQL | S2 (50 DTUs) | ~$75 USD |
| Cosmos DB | 400 RU/s | ~$23 USD |
| Blob Storage | LRS, 50 GB | ~$1 USD |
| Redis Cache | Standard C1 | ~$55 USD |
| Azure Functions | Consumption | ~$5 USD |
| Azure Maps | S0 | ~$10 USD |
| API Management | Developer | ~$50 USD |
| **Total Estimado** | | **~$444 USD/mês** |

> 💡 Em produção, usar **App Service Plan compartilhado** para as 3 APIs reduz o custo pela metade.

---

## 🛠️ Tecnologias

![Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)
![SQL](https://img.shields.io/badge/Azure_SQL-CC2927?style=for-the-badge&logo=microsoft-sql-server&logoColor=white)
![CosmosDB](https://img.shields.io/badge/Cosmos_DB-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)

---

## 📚 Referências

- [Azure App Service Docs](https://docs.microsoft.com/azure/app-service)
- [Azure Maps Documentation](https://docs.microsoft.com/azure/azure-maps)
- [Repositório Base DIO](https://github.com/digitalinnovationone/Microsoft_Application_Platform)
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture)

---

*⭐ Se este projeto foi útil, deixe uma estrela no repositório!*
