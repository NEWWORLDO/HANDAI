# Security-First AI Arbitrage System Implementation Guide

## **System architecture overview**

This implementation guide provides a complete blueprint for building a security-enhanced cryptocurrency arbitrage system that integrates SlowMist's security knowledge with Freqtrade and Hummingbot trading frameworks. The system prioritizes fail-closed security patterns, ensuring trades are blocked when security validation is unavailable.

### Core components
- **ETL Pipeline**: Processes SlowMist Knowledge-Base security documents
- **Weaviate Vector Database**: Stores and indexes security knowledge with hybrid search
- **FastAPI Security Service**: REST API providing security assessments
- **Freqtrade Plugin**: Pre-trade security validation integration
- **Hummingbot Controller**: Strategy V2 security-aware arbitrage implementation
- **Docker Compose**: Multi-service orchestration with proper networking
- **C# Wrapper Services**: Integration layer for C# developers

## 1. Docker Compose Infrastructure Setup

### Complete docker-compose.yml for the trading system

```yaml
version: '3.8'

services:
  # Weaviate Vector Database for Security Knowledge
  weaviate:
    image: cr.weaviate.io/semitechnologies/weaviate:1.30.4
    container_name: weaviate-security
    ports:
      - "8080:8080"
      - "50051:50051"
    volumes:
      - weaviate_data:/var/lib/weaviate
    environment:
      QUERY_DEFAULTS_LIMIT: 25
      AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: 'true'
      PERSISTENCE_DATA_PATH: '/var/lib/weaviate'
      ENABLE_API_BASED_MODULES: 'true'
      CLUSTER_HOSTNAME: 'node1'
      LIMIT_RESOURCES: 'true'
      GOMEMLIMIT: '10GiB'
      GOMAXPROCS: '4'
    networks:
      - trading-backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/v1/.well-known/ready"]
      interval: 30s
      timeout: 10s
      retries: 5

  # PostgreSQL for Trading Data
  postgres:
    image: postgres:15-alpine
    container_name: trading-postgres
    environment:
      POSTGRES_USER: trading_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: trading_data
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - trading-backend
    restart: unless-stopped

  # FastAPI Security Service
  security-api:
    build:
      context: ./security-service
      dockerfile: Dockerfile
    container_name: security-api
    environment:
      - WEAVIATE_URL=http://weaviate:8080
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_URL=redis://redis:6379
    ports:
      - "8001:8000"
    networks:
      - trading-backend
      - trading-frontend
    depends_on:
      - weaviate
      - redis
    restart: unless-stopped

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: redis-cache
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - trading-backend
    restart: unless-stopped

  # Freqtrade Bot with Security Plugin
  freqtrade:
    build:
      context: ./freqtrade
      dockerfile: Dockerfile
    container_name: freqtrade-secure
    volumes:
      - ./freqtrade/user_data:/freqtrade/user_data
    environment:
      - SECURITY_API_URL=http://security-api:8000
      - TRADING_MODE=${TRADING_MODE:-dry_run}
    ports:
      - "8002:8080"
    networks:
      - trading-backend
    depends_on:
      - security-api
    restart: unless-stopped

  # Hummingbot with Security Controller
  hummingbot:
    image: hummingbot/hummingbot:latest
    container_name: hummingbot-secure
    volumes:
      - ./hummingbot/conf:/home/hummingbot/conf
      - ./hummingbot/scripts:/home/hummingbot/scripts
    environment:
      - SECURITY_SERVICE_URL=http://security-api:8000
      - PAPER_TRADING=${PAPER_TRADING:-true}
    networks:
      - trading-backend
    depends_on:
      - security-api
    tty: true
    stdin_open: true
    restart: unless-stopped

volumes:
  weaviate_data:
  postgres_data:
  redis_data:

networks:
  trading-frontend:
    driver: bridge
  trading-backend:
    driver: bridge
    internal: true
```

## 2. ETL Pipeline for SlowMist Knowledge-Base

### Python ETL script to process security documents

```python
# etl/slowmist_etl_pipeline.py
import os
import git
import logging
from pathlib import Path
from typing import List, Dict
import frontmatter
import pdfplumber
from sentence_transformers import SentenceTransformer
import weaviate
from datetime import datetime

class SlowMistETLPipeline:
    def __init__(self, config: Dict):
        self.config = config
        self.logger = logging.getLogger(__name__)
        self.embedding_model = SentenceTransformer('all-MiniLM-L6-v2')
        self.weaviate_client = self._init_weaviate()
        
    def _init_weaviate(self):
        """Initialize Weaviate client with OpenAI embeddings"""
        return weaviate.Client(
            url=self.config['weaviate_url'],
            additional_headers={
                "X-OpenAI-Api-Key": self.config['openai_api_key']
            }
        )
    
    def clone_repository(self):
        """Clone or update SlowMist Knowledge-Base"""
        repo_path = self.config['local_repo_path']
        repo_url = "https://github.com/slowmist/Knowledge-Base.git"
        
        if os.path.exists(repo_path):
            repo = git.Repo(repo_path)
            repo.remotes.origin.pull()
            self.logger.info("Updated existing repository")
        else:
            git.Repo.clone_from(repo_url, repo_path)
            self.logger.info("Cloned repository")
    
    def extract_documents(self) -> List[Dict]:
        """Extract content from Markdown and PDF files"""
        documents = []
        repo_path = Path(self.config['local_repo_path'])
        
        # Process Markdown files
        for md_file in repo_path.rglob("*.md"):
            try:
                with open(md_file, 'r', encoding='utf-8') as f:
                    post = frontmatter.load(f)
                    
                doc = {
                    'content': post.content,
                    'title': post.metadata.get('title', md_file.stem),
                    'source': 'SlowMist Knowledge-Base',
                    'category': self._categorize_document(md_file),
                    'file_path': str(md_file),
                    'document_type': 'markdown'
                }
                documents.append(doc)
            except Exception as e:
                self.logger.error(f"Error processing {md_file}: {e}")
        
        # Process PDF files
        for pdf_file in repo_path.rglob("*.pdf"):
            try:
                with pdfplumber.open(pdf_file) as pdf:
                    text = ""
                    for page in pdf.pages:
                        text += page.extract_text() or ""
                    
                    doc = {
                        'content': text,
                        'title': pdf_file.stem,
                        'source': 'SlowMist Knowledge-Base',
                        'category': self._categorize_document(pdf_file),
                        'file_path': str(pdf_file),
                        'document_type': 'pdf'
                    }
                    documents.append(doc)
            except Exception as e:
                self.logger.error(f"Error processing {pdf_file}: {e}")
        
        return documents
    
    def _categorize_document(self, file_path: Path) -> str:
        """Categorize document based on file path"""
        path_str = str(file_path).lower()
        
        if 'vulnerability' in path_str or '漏洞' in path_str:
            return 'vulnerability'
        elif 'audit' in path_str or '审计' in path_str:
            return 'audit'
        elif 'attack' in path_str or '攻击' in path_str:
            return 'attack_analysis'
        else:
            return 'general_security'
    
    def chunk_documents(self, documents: List[Dict]) -> List[Dict]:
        """Split documents into chunks for embedding"""
        chunked_docs = []
        
        for doc in documents:
            chunks = self._split_text(doc['content'], chunk_size=512, overlap=50)
            
            for i, chunk in enumerate(chunks):
                chunked_doc = {
                    'id': f"{Path(doc['file_path']).stem}_{i}",
                    'content': chunk,
                    'title': doc['title'],
                    'source': doc['source'],
                    'category': doc['category'],
                    'chunk_index': i,
                    'created_date': datetime.utcnow().isoformat()
                }
                chunked_docs.append(chunked_doc)
        
        return chunked_docs
    
    def _split_text(self, text: str, chunk_size: int, overlap: int) -> List[str]:
        """Split text into overlapping chunks"""
        words = text.split()
        chunks = []
        
        for i in range(0, len(words), chunk_size - overlap):
            chunk = ' '.join(words[i:i + chunk_size])
            chunks.append(chunk)
        
        return chunks
    
    def run_pipeline(self):
        """Execute the complete ETL pipeline"""
        self.logger.info("Starting SlowMist ETL pipeline")
        
        # Clone/update repository
        self.clone_repository()
        
        # Extract documents
        documents = self.extract_documents()
        self.logger.info(f"Extracted {len(documents)} documents")
        
        # Chunk documents
        chunked_docs = self.chunk_documents(documents)
        self.logger.info(f"Created {len(chunked_docs)} document chunks")
        
        # Load into Weaviate (handled by separate script)
        return chunked_docs

# Configuration
config = {
    'weaviate_url': 'http://localhost:8080',
    'openai_api_key': os.getenv('OPENAI_API_KEY'),
    'local_repo_path': './data/slowmist-knowledge-base'
}

# Run ETL
pipeline = SlowMistETLPipeline(config)
documents = pipeline.run_pipeline()
```

## 3. Weaviate Schema and Data Loading

### Weaviate schema setup for security documents

```python
# weaviate/schema_setup.py
import weaviate
import weaviate.classes as wvc
from typing import List, Dict

class WeaviateSecuritySchema:
    def __init__(self, client: weaviate.Client):
        self.client = client
    
    def create_schema(self):
        """Create Weaviate schema for security documents"""
        
        security_documents = self.client.collections.create(
            name="SecurityDocument",
            vectorizer_config=wvc.config.Configure.Vectorizer.text2vec_openai(
                model="text-embedding-3-small",
                dimensions=1536,
                vectorize_collection_name=False
            ),
            
            properties=[
                wvc.config.Property(
                    name="content",
                    data_type=wvc.config.DataType.TEXT,
                    description="Document content for vector search",
                    vectorize_property_name=False,
                    index_filterable=True,
                    index_searchable=True
                ),
                wvc.config.Property(
                    name="title",
                    data_type=wvc.config.DataType.TEXT,
                    vectorize_property_name=False,
                    index_filterable=True
                ),
                wvc.config.Property(
                    name="category",
                    data_type=wvc.config.DataType.TEXT,
                    vectorize_property_name=False,
                    index_filterable=True
                ),
                wvc.config.Property(
                    name="source",
                    data_type=wvc.config.DataType.TEXT,
                    vectorize_property_name=False,
                    index_filterable=True
                ),
                wvc.config.Property(
                    name="severity",
                    data_type=wvc.config.DataType.TEXT,
                    vectorize_property_name=False,
                    index_filterable=True
                ),
                wvc.config.Property(
                    name="tags",
                    data_type=wvc.config.DataType.TEXT_ARRAY,
                    vectorize_property_name=False,
                    index_filterable=True
                ),
                wvc.config.Property(
                    name="created_date",
                    data_type=wvc.config.DataType.DATE,
                    vectorize_property_name=False,
                    index_filterable=True
                )
            ],
            
            inverted_index_config=wvc.config.Configure.inverted_index(
                bm25_b=0.75,
                bm25_k1=1.2,
                index_timestamps=True
            ),
            
            vector_index_config=wvc.config.Configure.VectorIndex.hnsw(
                distance_metric=wvc.config.VectorDistances.COSINE,
                ef_construction=128,
                ef=64,
                max_connections=32
            )
        )
        
        return security_documents
    
    def batch_import_documents(self, documents: List[Dict], batch_size: int = 100):
        """Batch import documents into Weaviate"""
        collection = self.client.collections.get("SecurityDocument")
        
        with collection.batch.fixed_size(batch_size=batch_size) as batch:
            for doc in documents:
                batch.add_object(
                    properties={
                        "content": doc["content"],
                        "title": doc["title"],
                        "category": doc["category"],
                        "source": doc["source"],
                        "severity": doc.get("severity", "medium"),
                        "tags": doc.get("tags", []),
                        "created_date": doc["created_date"]
                    }
                )
        
        print(f"Imported {len(documents)} documents")

# Initialize and setup
client = weaviate.connect_to_local(
    host="localhost",
    port=8080,
    headers={"X-OpenAI-Api-Key": os.getenv("OPENAI_API_KEY")}
)

schema_manager = WeaviateSecuritySchema(client)
schema_manager.create_schema()
```

## 4. FastAPI Security Service Implementation

### Complete FastAPI service with endpoints

```python
# security-service/app/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
import weaviate
from typing import List, Dict, Optional
from pydantic import BaseModel
import redis
import json
import logging

# Data models
class SecuritySearchRequest(BaseModel):
    query: str
    filters: Optional[Dict] = None
    limit: int = 10
    alpha: float = 0.7  # Hybrid search weight

class TokenRiskRequest(BaseModel):
    token_address: str
    chain: str
    protocol: Optional[str] = None

class SecurityResponse(BaseModel):
    status: str  # safe, warning, dangerous, blocked
    risk_score: float
    risk_factors: List[str]
    recommendations: List[str]
    confidence: float

# Application lifecycle
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    app.state.weaviate_client = weaviate.connect_to_local(
        host="weaviate",
        port=8080,
        headers={"X-OpenAI-Api-Key": os.getenv("OPENAI_API_KEY")}
    )
    
    app.state.redis_client = redis.from_url(
        os.getenv("REDIS_URL", "redis://redis:6379"),
        decode_responses=True
    )
    
    yield
    
    # Shutdown
    app.state.weaviate_client.close()
    app.state.redis_client.close()

# Initialize FastAPI
app = FastAPI(
    title="Trading Security Knowledge API",
    version="1.0.0",
    lifespan=lifespan
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["GET", "POST"],
    allow_headers=["*"]
)

@app.post("/api/v1/security/search")
async def search_security_knowledge(request: SecuritySearchRequest):
    """Search security knowledge base with hybrid search"""
    try:
        # Check cache first
        cache_key = f"search:{request.query}:{request.limit}"
        cached = app.state.redis_client.get(cache_key)
        if cached:
            return json.loads(cached)
        
        # Query Weaviate
        collection = app.state.weaviate_client.collections.get("SecurityDocument")
        
        response = collection.query.hybrid(
            query=request.query,
            limit=request.limit,
            alpha=request.alpha,
            return_properties=["content", "title", "category", "severity", "tags"]
        )
        
        results = []
        for obj in response.objects:
            results.append({
                "content": obj.properties["content"],
                "title": obj.properties["title"],
                "category": obj.properties["category"],
                "severity": obj.properties["severity"],
                "score": obj.metadata.score if hasattr(obj.metadata, 'score') else 0
            })
        
        # Cache results
        app.state.redis_client.setex(
            cache_key, 
            300,  # 5 minutes TTL
            json.dumps(results)
        )
        
        return {"results": results, "total": len(results)}
        
    except Exception as e:
        logging.error(f"Search error: {e}")
        # Fail-closed behavior
        raise HTTPException(status_code=503, detail="Security service unavailable")

@app.post("/api/v1/risk/assess/token", response_model=SecurityResponse)
async def assess_token_risk(request: TokenRiskRequest):
    """Assess security risk for a specific token"""
    try:
        # Search for security issues related to this token
        search_query = f"{request.token_address} {request.chain} vulnerability exploit"
        
        collection = app.state.weaviate_client.collections.get("SecurityDocument")
        response = collection.query.hybrid(
            query=search_query,
            limit=5,
            alpha=0.8
        )
        
        risk_factors = []
        risk_score = 0.0
        
        # Analyze search results
        for obj in response.objects:
            if "vulnerability" in obj.properties.get("category", ""):
                risk_factors.append("Known vulnerabilities found")
                risk_score += 30
            if "exploit" in obj.properties.get("tags", []):
                risk_factors.append("Previous exploits recorded")
                risk_score += 40
            if obj.properties.get("severity") == "critical":
                risk_factors.append("Critical security issues")
                risk_score += 30
        
        # Determine status
        if risk_score >= 70:
            status = "blocked"
        elif risk_score >= 50:
            status = "dangerous"
        elif risk_score >= 30:
            status = "warning"
        else:
            status = "safe"
        
        recommendations = []
        if status in ["blocked", "dangerous"]:
            recommendations.append("Avoid trading this token")
            recommendations.append("Conduct thorough security audit")
        elif status == "warning":
            recommendations.append("Trade with caution")
            recommendations.append("Use smaller position sizes")
        
        return SecurityResponse(
            status=status,
            risk_score=min(risk_score, 100),
            risk_factors=risk_factors,
            recommendations=recommendations,
            confidence=0.85 if response.objects else 0.5
        )
        
    except Exception as e:
        logging.error(f"Risk assessment error: {e}")
        # Fail-closed: return maximum risk
        return SecurityResponse(
            status="blocked",
            risk_score=100,
            risk_factors=["Security assessment unavailable"],
            recommendations=["Do not trade - security check failed"],
            confidence=0.0
        )

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    try:
        # Check Weaviate connection
        app.state.weaviate_client.is_ready()
        
        # Check Redis connection
        app.state.redis_client.ping()
        
        return {"status": "healthy", "services": {"weaviate": "up", "redis": "up"}}
    except:
        return {"status": "unhealthy"}, 503
```

## 5. Freqtrade Security Plugin Implementation

### Custom Freqtrade strategy with security integration

```python
# freqtrade/user_data/strategies/SecurityAwareStrategy.py
from freqtrade.strategy import IStrategy
from pandas import DataFrame
import requests
import logging
from datetime import datetime
from typing import Dict, Optional

logger = logging.getLogger(__name__)

class SecurityAwareStrategy(IStrategy):
    """
    Freqtrade strategy with integrated security checks
    """
    
    # Strategy parameters
    minimal_roi = {"0": 0.10}
    stoploss = -0.05
    timeframe = '5m'
    
    # Security configuration
    security_api_url = "http://security-api:8000/api/v1"
    security_timeout = 5
    fail_closed = True
    max_risk_score = 50
    
    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """Add technical indicators"""
        # Add your indicators here
        dataframe['sma_20'] = dataframe['close'].rolling(window=20).mean()
        dataframe['sma_50'] = dataframe['close'].rolling(window=50).mean()
        
        return dataframe
    
    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """Define entry signals"""
        dataframe.loc[
            (
                (dataframe['sma_20'] > dataframe['sma_50']) &
                (dataframe['volume'] > 0)
            ),
            'enter_long'] = 1
        
        return dataframe
    
    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """Define exit signals"""
        dataframe.loc[
            (
                (dataframe['sma_20'] < dataframe['sma_50']) &
                (dataframe['volume'] > 0)
            ),
            'exit_long'] = 1
        
        return dataframe
    
    def confirm_trade_entry(self, pair: str, order_type: str, amount: float,
                          rate: float, time_in_force: str, current_time: datetime,
                          entry_tag: Optional[str], side: str, **kwargs) -> bool:
        """
        Security check before trade entry
        """
        try:
            # Extract token addresses from pair
            base_token, quote_token = pair.split('/')
            
            # Call security API
            response = requests.post(
                f"{self.security_api_url}/risk/assess/token",
                json={
                    "token_address": base_token,
                    "chain": "ethereum",  # Adjust based on exchange
                    "protocol": "spot"
                },
                timeout=self.security_timeout
            )
            
            if response.status_code == 200:
                result = response.json()
                
                # Check risk score
                if result['risk_score'] > self.max_risk_score:
                    logger.warning(
                        f"Trade blocked for {pair}: Risk score {result['risk_score']} "
                        f"exceeds threshold {self.max_risk_score}"
                    )
                    return False
                
                # Check status
                if result['status'] in ['blocked', 'dangerous']:
                    logger.warning(
                        f"Trade blocked for {pair}: Security status is {result['status']}"
                    )
                    return False
                
                logger.info(f"Security check passed for {pair}: {result['status']}")
                return True
            
            else:
                logger.error(f"Security API error: {response.status_code}")
                return not self.fail_closed
                
        except Exception as e:
            logger.error(f"Security check failed: {e}")
            # Fail-closed behavior
            return not self.fail_closed
    
    def custom_exit(self, pair: str, trade: 'Trade', current_time: datetime,
                    current_rate: float, current_profit: float, **kwargs) -> Optional[str]:
        """
        Emergency exit on security alerts
        """
        try:
            # Check for security alerts
            response = requests.post(
                f"{self.security_api_url}/risk/assess/token",
                json={
                    "token_address": pair.split('/')[0],
                    "chain": "ethereum"
                },
                timeout=2  # Shorter timeout for exit checks
            )
            
            if response.status_code == 200:
                result = response.json()
                if result['status'] == 'blocked':
                    return 'security_emergency_exit'
                    
        except Exception as e:
            logger.error(f"Exit security check failed: {e}")
        
        return None
```

### Freqtrade configuration with security plugin

```json
{
    "max_open_trades": 3,
    "stake_currency": "USDT",
    "stake_amount": "unlimited",
    "tradable_balance_ratio": 0.99,
    "dry_run": true,
    "dry_run_wallet": 10000,
    
    "exchange": {
        "name": "binance",
        "key": "",
        "secret": "",
        "ccxt_config": {
            "enableRateLimit": true
        },
        "pair_whitelist": [
            "BTC/USDT",
            "ETH/USDT"
        ]
    },
    
    "strategy": "SecurityAwareStrategy",
    
    "telegram": {
        "enabled": false
    },
    
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",
        "listen_port": 8080,
        "verbosity": "error",
        "jwt_secret_key": "your-secret-key",
        "CORS_origins": ["http://localhost:3000"],
        "username": "freqtrader",
        "password": "password"
    }
}
```

## 6. Hummingbot Security Controller Implementation

### Strategy V2 security-aware arbitrage controller

```python
# hummingbot/scripts/security_arbitrage_controller.py
import asyncio
from decimal import Decimal
from typing import List, Dict, Optional
import aiohttp
from hummingbot.strategy_v2.controllers.controller_base import ControllerBase
from hummingbot.strategy_v2.models.executor_actions import CreateExecutorAction

class SecurityAwareArbitrageConfig:
    min_profitability: Decimal = Decimal("0.01")
    order_amount: Decimal = Decimal("100")
    security_service_url: str = "http://security-api:8000"
    max_risk_score: int = 50
    fail_closed: bool = True

class SecurityAwareArbitrageController(ControllerBase):
    def __init__(self, config: SecurityAwareArbitrageConfig):
        super().__init__(config)
        self.config = config
        self._security_session = None
        
    async def start(self):
        await super().start()
        self._security_session = aiohttp.ClientSession()
        
    async def stop(self):
        if self._security_session:
            await self._security_session.close()
        await super().stop()
    
    async def check_token_security(self, token: str) -> Dict:
        """Check token security before arbitrage"""
        try:
            async with self._security_session.post(
                f"{self.config.security_service_url}/api/v1/risk/assess/token",
                json={"token_address": token, "chain": "ethereum"},
                timeout=aiohttp.ClientTimeout(total=5)
            ) as response:
                if response.status == 200:
                    return await response.json()
                else:
                    return {"status": "blocked", "risk_score": 100}
        except Exception as e:
            self.logger().error(f"Security check failed: {e}")
            return {"status": "blocked", "risk_score": 100} if self.config.fail_closed else {"status": "warning", "risk_score": 50}
    
    async def create_arbitrage_opportunity(self, buying_market: str, selling_market: str) -> Optional[CreateExecutorAction]:
        """Create arbitrage executor with security validation"""
        
        # Extract token from market pair
        token = buying_market.split('_')[1].split('-')[0]
        
        # Security check
        security_result = await self.check_token_security(token)
        
        if security_result['risk_score'] > self.config.max_risk_score:
            self.logger().warning(f"Arbitrage blocked for {token}: Risk score {security_result['risk_score']}")
            return None
        
        if security_result['status'] in ['blocked', 'dangerous']:
            self.logger().warning(f"Arbitrage blocked for {token}: Status {security_result['status']}")
            return None
        
        # Proceed with arbitrage if security check passes
        return CreateExecutorAction(
            controller_id=self.config.id,
            executor_config={
                "buying_market": buying_market,
                "selling_market": selling_market,
                "order_amount": self.config.order_amount,
                "min_profitability": self.config.min_profitability
            }
        )
    
    def determine_executor_actions(self) -> List[CreateExecutorAction]:
        """Main control loop"""
        actions = []
        
        # Example: Check BTC arbitrage opportunity
        btc_action = asyncio.create_task(
            self.create_arbitrage_opportunity(
                "binance_BTC-USDT",
                "kucoin_BTC-USDT"
            )
        )
        
        if btc_action.result():
            actions.append(btc_action.result())
        
        return actions
```

## 7. C# Wrapper Service Implementation

### C# client for consuming Python REST APIs

```csharp
// CSharpWrapper/Services/SecurityTradingService.cs
using System;
using System.Net.Http;
using System.Text;
using System.Text.Json;
using System.Threading.Tasks;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;

namespace TradingBot.Services
{
    public class SecurityTradingService
    {
        private readonly HttpClient _httpClient;
        private readonly ILogger<SecurityTradingService> _logger;
        private readonly string _securityApiUrl;
        private readonly int _maxRiskThreshold;

        public SecurityTradingService(
            HttpClient httpClient, 
            IConfiguration configuration,
            ILogger<SecurityTradingService> logger)
        {
            _httpClient = httpClient;
            _logger = logger;
            _securityApiUrl = configuration["SecurityApi:BaseUrl"];
            _maxRiskThreshold = configuration.GetValue<int>("SecurityApi:MaxRiskThreshold", 50);
        }

        public async Task<bool> ValidateTradeSecurityAsync(string tokenAddress, string chain)
        {
            try
            {
                var request = new
                {
                    token_address = tokenAddress,
                    chain = chain
                };

                var json = JsonSerializer.Serialize(request);
                var content = new StringContent(json, Encoding.UTF8, "application/json");

                var response = await _httpClient.PostAsync(
                    $"{_securityApiUrl}/api/v1/risk/assess/token", 
                    content);

                if (response.IsSuccessStatusCode)
                {
                    var responseContent = await response.Content.ReadAsStringAsync();
                    var result = JsonSerializer.Deserialize<SecurityAssessmentResponse>(responseContent);

                    // Fail-closed decision logic
                    if (result.RiskScore > _maxRiskThreshold)
                    {
                        _logger.LogWarning($"Trade blocked: Risk score {result.RiskScore} exceeds threshold");
                        return false;
                    }

                    if (result.Status == "blocked" || result.Status == "dangerous")
                    {
                        _logger.LogWarning($"Trade blocked: Security status is {result.Status}");
                        return false;
                    }

                    return true;
                }
                else
                {
                    _logger.LogError($"Security API returned {response.StatusCode}");
                    // Fail-closed on API errors
                    return false;
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Security validation failed");
                // Fail-closed on exceptions
                return false;
            }
        }

        public async Task<SecurityKnowledgeResult> SearchSecurityKnowledgeAsync(string query)
        {
            try
            {
                var request = new { query = query, limit = 10 };
                var json = JsonSerializer.Serialize(request);
                var content = new StringContent(json, Encoding.UTF8, "application/json");

                var response = await _httpClient.PostAsync(
                    $"{_securityApiUrl}/api/v1/security/search", 
                    content);

                if (response.IsSuccessStatusCode)
                {
                    var responseContent = await response.Content.ReadAsStringAsync();
                    return JsonSerializer.Deserialize<SecurityKnowledgeResult>(responseContent);
                }
                else
                {
                    return new SecurityKnowledgeResult 
                    { 
                        Results = new List<SecurityDocument>(),
                        ErrorMessage = "Security knowledge service unavailable"
                    };
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Security knowledge search failed");
                return new SecurityKnowledgeResult 
                { 
                    Results = new List<SecurityDocument>(),
                    ErrorMessage = "Search failed - proceed with caution"
                };
            }
        }
    }

    // Response models
    public class SecurityAssessmentResponse
    {
        public string Status { get; set; }
        public double RiskScore { get; set; }
        public List<string> RiskFactors { get; set; }
        public List<string> Recommendations { get; set; }
        public double Confidence { get; set; }
    }

    public class SecurityKnowledgeResult
    {
        public List<SecurityDocument> Results { get; set; }
        public int Total { get; set; }
        public string ErrorMessage { get; set; }
    }

    public class SecurityDocument
    {
        public string Content { get; set; }
        public string Title { get; set; }
        public string Category { get; set; }
        public string Severity { get; set; }
        public double Score { get; set; }
    }
}
```

### C# API wrapper Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["TradingBot.csproj", "."]
RUN dotnet restore "TradingBot.csproj"
COPY . .
RUN dotnet build "TradingBot.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "TradingBot.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "TradingBot.dll"]
```

## 8. Step-by-Step Setup Guide

### Prerequisites
- Docker and Docker Compose installed
- OpenAI API key for embeddings
- Exchange API keys (for live trading)
- Python 3.11+ (for local development)
- .NET 8 SDK (for C# wrapper)

### Setup Steps

1. **Clone the repository structure**
```bash
mkdir security-trading-system
cd security-trading-system
mkdir -p {etl,weaviate,security-service,freqtrade,hummingbot,csharp-wrapper}
```

2. **Set up environment variables**
```bash
# Create .env file
cat > .env << EOF
OPENAI_API_KEY=your-openai-api-key
POSTGRES_PASSWORD=secure-password
TRADING_MODE=dry_run
PAPER_TRADING=true
EOF
```

3. **Build and start the infrastructure**
```bash
# Start all services
docker-compose up -d

# Check service health
docker-compose ps
```

4. **Initialize Weaviate schema**
```bash
# Run schema setup
docker exec -it security-trading-system_weaviate_1 python /app/schema_setup.py
```

5. **Run ETL pipeline to load SlowMist data**
```bash
# Execute ETL pipeline
docker exec -it security-trading-system_etl_1 python /app/slowmist_etl_pipeline.py
```

6. **Configure and start trading bots**
```bash
# Freqtrade
docker exec -it freqtrade-secure freqtrade trade --strategy SecurityAwareStrategy

# Hummingbot
docker exec -it hummingbot-secure ./start
```

7. **Test the security API**
```bash
# Health check
curl http://localhost:8001/health

# Test security search
curl -X POST http://localhost:8001/api/v1/security/search \
  -H "Content-Type: application/json" \
  -d '{"query": "smart contract vulnerability", "limit": 5}'

# Test risk assessment
curl -X POST http://localhost:8001/api/v1/risk/assess/token \
  -H "Content-Type: application/json" \
  -d '{"token_address": "0x1234...", "chain": "ethereum"}'
```

## 9. Security and Reliability Considerations

### Fail-Closed Architecture
- All components default to blocking trades when security checks fail
- Network timeouts trigger conservative responses
- Service unavailability prevents trade execution

### Error Handling Strategy
```python
# Example fail-closed pattern
try:
    security_result = await check_security(token)
    if security_result['safe']:
        execute_trade()
except Exception:
    # Always block on errors
    block_trade()
    log_security_failure()
```

### Monitoring and Logging
- Structured logging with correlation IDs
- Prometheus metrics for service health
- Alert on security check failures
- Trade decision audit trail

### Paper Trading Configuration
- Always start with paper trading enabled
- Test security integrations thoroughly
- Validate fail-closed behavior
- Monitor false positive rates

## 10. Production Deployment Checklist

### Security Hardening
- [ ] Enable Weaviate authentication
- [ ] Implement API rate limiting
- [ ] Use secrets management for keys
- [ ] Enable TLS for all services
- [ ] Implement request signing

### Performance Optimization
- [ ] Configure Redis caching TTLs
- [ ] Optimize Weaviate vector indices
- [ ] Implement connection pooling
- [ ] Set appropriate resource limits

### Monitoring Setup
- [ ] Configure Prometheus exporters
- [ ] Set up Grafana dashboards
- [ ] Implement health check alerts
- [ ] Create runbooks for incidents

### Backup and Recovery
- [ ] Regular Weaviate backups
- [ ] PostgreSQL replication
- [ ] Configuration version control
- [ ] Disaster recovery procedures

## Summary

This implementation provides a comprehensive security-first arbitrage trading system that:

1. **Processes security knowledge** from SlowMist Knowledge-Base using an automated ETL pipeline
2. **Stores and indexes** security data in Weaviate with hybrid search capabilities
3. **Provides REST APIs** for security assessment and knowledge retrieval
4. **Integrates security checks** into both Freqtrade and Hummingbot trading frameworks
5. **Implements fail-closed patterns** throughout to ensure safety when services are degraded
6. **Offers C# wrapper services** for seamless integration with existing C# codebases

The system prioritizes security and reliability while maintaining the flexibility needed for cryptocurrency arbitrage trading. All components are containerized for easy deployment and scaling.