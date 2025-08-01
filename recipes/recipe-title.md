---
title: Recipe Title
description: Recipe Description
hidden: false
recipe:
  color: '#018FF4'
  icon: 🦉
---
```python Python
# Internal backend components are separated into files to indicate their logical
# responsibilities and enhance modularity. However, for a single recipe panel,
# they are presented consecutively within one continuous block as requested.

# --- Code structure breakdown in file system: ---
# nexusmind-ai/
# ├── src/
# │   ├── main.py
# │   ├── config.py
# │   ├── database.py
# │   ├── models.py
# │   ├── schemas.py
# │   ├── api/
# │   │   ├── __init__.py
# │   │   ├── routers.py
# │   │   └── v1/
# │   │       ├── __init__.py
# │   │       └── file_contents.py  # Endpoints implemented above
# │   ├── services/
# │   │   ├── __init__.py
# │   │   ├── file_ingestion.py     # Content included above
# │   │   └── ai_analysis.py        # Content included above
# │   ├── tasks/
# │   │   ├── __init__.py
# │   │   └── celery_tasks.py       # Content included above
# │   └── worker.py                  # Placeholder, just `from src.tasks.celery_tasks import celery_app; celery_app.worker_main(['worker', '--loglevel=info'])`
# └── alembic_migrations/ # Or similar dir for your Alembic setup
# ============================================================================
# FILE: nexusmind-ai/src/config.py
# DESCRIPTION: Core application settings loaded from environment variables.
# ============================================================================
import os
from dotenv import load_dotenv

# Load environment variables from .env file at startup
load_dotenv()

class Settings:
    """
    Configuration settings for the NexusMind AI application, loaded from
    environment variables. This ensures sensitive data and deployment-specific
    parameters are externalized.
    """
    # Database connection URL
    DATABASE_URL: str = os.getenv("DATABASE_URL", "postgresql+asyncpg://user:password@localhost:5432/nexusmind_db")
    
    # Application host and port
    APP_HOST: str = os.getenv("APP_HOST", "0.0.0.0")
    APP_PORT: int = int(os.getenv("APP_PORT", 8000))
    
    # Celery (Asynchronous Task Queue) broker and result backend URLs
    CELERY_BROKER_URL: str = os.getenv("CELERY_BROKER_URL", "redis://localhost:6379/0")
    CELERY_RESULT_BACKEND: str = os.getenv("CELERY_RESULT_BACKEND", "redis://localhost:6379/0")
    
    # Path for local file storage (for uploaded content)
    FILE_STORAGE_PATH: str = os.getenv("FILE_STORAGE_PATH", "./local_file_storage")

    def __init__(self):
        """
        Initializes settings and ensures necessary local directories exist.
        In a production environment, file storage would likely be a cloud
        blob storage service (e.g., S3, GCS).
        """
        # Create the local file storage directory if it doesn't exist
        if not os.path.exists(self.FILE_STORAGE_PATH):
            os.makedirs(self.FILE_STORAGE_PATH)
            print(f"Created local file storage directory: {self.FILE_STORAGE_PATH}")
        else:
            print(f"Local file storage directory exists: {self.FILE_STORAGE_PATH}")

# Instantiate settings globally for easy access throughout the application
settings = Settings()

# Basic validation to ensure critical environment variables are set
if not settings.DATABASE_URL:
    raise ValueError("DATABASE_URL environment variable is not set. Please check your .env file.")


# ============================================================================
# FILE: nexusmind-ai/src/database.py
# DESCRIPTION: Sets up SQLAlchemy for asynchronous database connections.
# ============================================================================
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase, sessionmaker
# from src.config import settings # Config is usually loaded globally once

# --- Asynchronous Database Engine Setup ---
# `create_async_engine` establishes the connection pool for PostgreSQL using asyncpg.
# `echo=False` suppresses SQL query logging; set to `True` for debugging.
# `pool_size` and `max_overflow` manage the connection pool.
engine = create_async_engine(
    settings.DATABASE_URL,
    echo=False,
    pool_size=10,
    max_overflow=20
)

# --- Asynchronous Session Management ---
# `async_sessionmaker` creates a factory for producing new `AsyncSession` objects.
# Each `AsyncSession` manages a single conversation with the database.
# `expire_on_commit=False` allows ORM objects to remain valid and usable after a session commit.
AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False
)

# --- Declarative Base for ORM Models ---
# `DeclarativeBase` is the fundamental class that all SQLAlchemy ORM models in `src/models.py`
# will inherit from. It connects Python classes to database tables.
class Base(DeclarativeBase):
    pass

# --- FastAPI Dependency for Database Session Injection ---
# This asynchronous generator function serves as a FastAPI dependency.
# It ensures that each API request gets its own database session, which is
# properly opened at the start of the request and closed (or rolled back) at its end.
async def get_db() -> AsyncSession:
    """
    FastAPI dependency that yields an asynchronous database session.
    Manages session lifecycle (opening, yielding, and closing/rollback).
    """
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close() # Ensure session is closed

# --- Synchronous Engine for Alembic Migrations ---
# Alembic (our migration tool) often works better with a synchronous engine
# for its initial setup and command execution. This provides a temporary synchronous
# binding from the async engine's capabilities.
# NOTE: This part is for local Alembic setup ONLY and is not directly used by the running FastAPI app.
sync_engine = create_async_engine(settings.DATABASE_URL, echo=False)
SyncSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=sync_engine.sync_engine)


# ============================================================================
# FILE: nexusmind-ai/src/models.py
# DESCRIPTION: SQLAlchemy ORM Models defining the database schema.
# ============================================================================
from sqlalchemy import (
    Column, String, DateTime, JSON, Text, ForeignKey, Integer
)
from sqlalchemy.orm import relationship, Mapped, mapped_column
from sqlalchemy.dialects.postgresql import UUID # PostgreSQL-specific UUID type
import uuid # For default UUID generation
from datetime import datetime # For default datetime generation
from typing import List, Dict, Any, Optional

# from src.database import Base # Already conceptually combined
# from sqlalchemy.ext.declarative import declarative_base # Replaced by DeclarativeBase from src.database

# Base mixin for common ORM columns across all models,
# encapsulating default ID and timestamp generation.
class BaseModelMixin:
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=datetime.utcnow, onupdate=datetime.utcnow)

class FileContent(Base, BaseModelMixin):
    """
    SQLAlchemy ORM model representing a digital file uploaded to NexusMind AI.
    Stores file metadata, storage location, processing status, and links to
    semantic analysis results in the ledger.
    """
    __tablename__ = "file_contents"

    name: Mapped[str] = mapped_column(String(255), index=True) # User-provided file name
    file_type: Mapped[str] = mapped_column(String(100)) # MIME type (e.g., "application/pdf")
    size_bytes: Mapped[int] = mapped_column(Integer) # Size of the file in bytes
    external_id: Mapped[Optional[str]] = mapped_column(String(180), unique=True, nullable=True, index=True) # Custom ID for external systems
    status: Mapped[str] = mapped_column(String(50), default="uploaded") # Lifecycle status: "uploaded", "analysis_complete", etc.
    storage_path: Mapped[str] = mapped_column(Text, unique=True) # Internal path or key (e.g., S3 object key)
    
    # Link to the dedicated SemanticLedgerAccount (Content Account) for this file
    semantic_ledger_account_id: Mapped[Optional[uuid.UUID]] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey('semantic_ledger_accounts.id', ondelete='SET NULL'), # Soft link, keep file even if account deleted
        nullable=True, unique=True, index=True
    )
    # Link to the most recent AnalysisJob for this file
    latest_analysis_job_id: Mapped[Optional[uuid.UUID]] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey('analysis_jobs.id', ondelete='SET NULL'),
        nullable=True, index=True
    )
    # Flexible JSONB field for arbitrary user-defined metadata (using '_' to avoid keyword conflict)
    metadata_: Mapped[Optional[Dict[str, str]]] = mapped_column(JSON(none_as_null=True), nullable=True)

    # SQLAlchemy ORM Relationships for easier querying across linked models
    semantic_ledger_account_rel: Mapped[Optional["SemanticLedgerAccount"]] = relationship(
        "SemanticLedgerAccount",
        back_populates="file_content_rel",
        foreign_keys=[semantic_ledger_account_id],
        uselist=False # Indicates one-to-one or zero-to-one relationship
    )
    analysis_jobs: Mapped[List["AnalysisJob"]] = relationship(
        "AnalysisJob",
        back_populates="file_content_rel",
        order_by="desc(AnalysisJob.created_at)" # Order jobs by creation date, newest first
    )

class AnalysisProfile(Base, BaseModelMixin):
    """
    SQLAlchemy ORM model defining a reusable configuration for AI content analysis.
    Specifies which file types it applies to and a sequence of AI analysis steps.
    """
    __tablename__ = "analysis_profiles"

    name: Mapped[str] = mapped_column(String(255), unique=True, index=True) # Unique name for the profile
    description: Mapped[Optional[str]] = mapped_column(Text, nullable=True) # Detailed description
    target_file_types: Mapped[List[str]] = mapped_column(JSON(none_as_null=True)) # List of MIME types this profile handles
    analysis_steps: Mapped[List[Dict[str, Any]]] = mapped_column(JSON(none_as_null=True)) # Structured JSON list of steps (type, model, params)
    status: Mapped[str] = mapped_column(String(50), default="active") # "active", "archived"
    version: Mapped[int] = mapped_column(Integer, default=1) # Version tracking for profile changes
    metadata_: Mapped[Optional[Dict[str, str]]] = mapped_column(JSON(none_as_null=True), nullable=True)

class AnalysisJob(Base, BaseModelMixin):
    """
    SQLAlchemy ORM model representing a single execution instance of an `AnalysisProfile`
    on a specific `FileContent`. Tracks job status, start/end times, and any errors.
    """
    __tablename__ = "analysis_jobs"

    file_content_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey('file_contents.id', ondelete='CASCADE'), # Deleting file cascades to its jobs
        index=True
    )
    analysis_profile_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey('analysis_profiles.id', ondelete='SET NULL') # Jobs can persist even if profile is deleted
    )
    status: Mapped[str] = mapped_column(String(50), default="queued") # "queued", "processing", "completed", "failed", "cancelled"
    started_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True), nullable=True)
    completed_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True), nullable=True)
    error_details: Mapped[Optional[Dict[str, Any]]] = mapped_column(JSON(none_as_null=True), nullable=True) # Detailed error info (code, message, etc.)
    
    # Link to the generated SemanticLedgerTransaction upon job completion
    semantic_ledger_transaction_id: Mapped[Optional[uuid.UUID]] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey('semantic_ledger_transactions.id', ondelete='SET NULL'),
        nullable=True, unique=True
    )
    metadata_: Mapped[Optional[Dict[str, str]]] = mapped_column(JSON(none_as_null=True), nullable=True)

    file_content_rel: Mapped["FileContent"] = relationship(
        "FileContent",
        back_populates="analysis_jobs",
        foreign_keys=[file_content_id]
    )
    analysis_profile_rel: Mapped["AnalysisProfile"] = relationship(
        "AnalysisProfile",
        foreign_keys=[analysis_profile_id]
    )
    semantic_ledger_transaction_rel: Mapped[Optional["SemanticLedgerTransaction"]] = relationship(
        "SemanticLedgerTransaction",
        back_populates="analysis_job_rel",
        foreign_keys=[semantic_ledger_transaction_id],
        uselist=False
    )

class SemanticLedgerAccount(Base, BaseModelMixin):
    """
    SQLAlchemy ORM model representing an "account" in the semantic ledger.
    This can be a:
    - 'File Content Account' (dedicated to a specific FileContent)
    - 'Semantic Concept Account' (for abstract semantic topics, entities, etc.)
    """
    __tablename__ = "semantic_ledger_accounts"

    name: Mapped[str] = mapped_column(String(255), index=True)
    description: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    external_id: Mapped[str] = mapped_column(String(255), unique=True, index=True) # e.g., `FileContent.id` or `CONCEPT::AI_Ethics`
    type: Mapped[str] = mapped_column(String(50), index=True) # "file_content_account", "semantic_concept_account"
    
    # Link specifically to a FileContent if this is a file_content_account
    related_file_content_id: Mapped[Optional[uuid.UUID]] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey('file_contents.id', ondelete='CASCADE'), # Deleting FileContent cascades to its unique SLA
        nullable=True, unique=True, index=True
    )
    # Link to the most recent Insight Transaction that updated this account's understanding
    latest_insight_transaction_id: Mapped[Optional[uuid.UUID]] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey('semantic_ledger_transactions.id', ondelete='SET NULL'),
        nullable=True
    )
    metadata_: Mapped[Optional[Dict[str, str]]] = mapped_column(JSON(none_as_null=True), nullable=True)

    file_content_rel: Mapped[Optional["FileContent"]] = relationship(
        "FileContent",
        back_populates="semantic_ledger_account_rel",
        foreign_keys=[related_file_content_id],
        uselist=False
    )
    # Semantic entries belonging to this account (could be 'in-flow' to concepts, or direct for file)
    semantic_ledger_entries: Mapped[List["SemanticLedgerEntry"]] = relationship(
        "SemanticLedgerEntry",
        back_populates="ledger_account_rel",
        order_by="SemanticLedgerEntry.created_at"
    )

class SemanticLedgerTransaction(Base, BaseModelMixin):
    """
    SQLAlchemy ORM model representing an immutable 'Insight Transaction'.
    This is an atomic record of an AI analysis run's results on a specific `FileContent`.
    All `SemanticLedgerEntry` objects generated from one analysis job belong to one transaction.
    """
    __tablename__ = "semantic_ledger_transactions"

    description: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    status: Mapped[str] = mapped_column(String(50), default="posted") # Status for a finalized transaction is "posted"
    external_id: Mapped[str] = mapped_column(String(255), unique=True, index=True) # Linked to AnalysisJob ID
    file_content_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey('file_contents.id', ondelete='CASCADE')
    )
    analysis_profile_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey('analysis_profiles.id', ondelete='SET NULL')
    )
    number_of_semantic_entries: Mapped[int] = mapped_column(Integer, default=0) # Count of entries within this transaction
    processing_time_ms: Mapped[Optional[int]] = mapped_column(Integer, nullable=True) # How long the AI analysis took
    metadata_: Mapped[Optional[Dict[str, str]]] = mapped_column(JSON(none_as_null=True), nullable=True)

    analysis_job_rel: Mapped[Optional["AnalysisJob"]] = relationship(
        "AnalysisJob",
        back_populates="semantic_ledger_transaction_rel",
        primaryjoin="foreign(SemanticLedgerTransaction.external_id) == remote(AnalysisJob.id)",
        uselist=False # One-to-one from job -> transaction
    )
    file_content_rel: Mapped["FileContent"] = relationship("FileContent", foreign_keys=[file_content_id])
    analysis_profile_rel: Mapped["AnalysisProfile"] = relationship("AnalysisProfile", foreign_keys=[analysis_profile_id])
    semantic_entries: Mapped[List["SemanticLedgerEntry"]] = relationship(
        "SemanticLedgerEntry",
        back_populates="semantic_transaction_rel"
    )

class SemanticLedgerEntry(Base, BaseModelMixin):
    """
    SQLAlchemy ORM model representing an atomic, auditable semantic fact extracted
    from a FileContent. Each entry is a single piece of AI-derived insight.
    """
    __tablename__ = "semantic_ledger_entries"

    # semantic_data holds the actual structured insight (e.g., keyword, entity, sentiment score)
    semantic_data: Mapped[Dict[str, Any]] = mapped_column(JSON(none_as_null=True))
    file_content_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey('file_contents.id', ondelete='CASCADE'), # Delete entries if source file is deleted
        index=True
    )
    # The ledger_account_id is usually the 'Content Account' of the file, or a 'Concept Account'
    ledger_account_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey('semantic_ledger_accounts.id', ondelete='CASCADE'), # Delete entries if linked account is deleted
        index=True
    )
    # Link to the specific SemanticLedgerTransaction (analysis run) that created this entry
    semantic_transaction_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey('semantic_ledger_transactions.id', ondelete='CASCADE'), # Delete entries if parent transaction deleted
        index=True
    )
    metadata_: Mapped[Optional[Dict[str, str]]] = mapped_column(JSON(none_as_null=True), nullable=True)

    file_content_rel: Mapped["FileContent"] = relationship("FileContent", foreign_keys=[file_content_id])
    ledger_account_rel: Mapped["SemanticLedgerAccount"] = relationship("SemanticLedgerAccount", back_populates="semantic_ledger_entries")
    semantic_transaction_rel: Mapped["SemanticLedgerTransaction"] = relationship("SemanticLedgerTransaction", back_populates="semantic_entries")


# ============================================================================
# FILE: nexusmind-ai/src/schemas.py
# DESCRIPTION: Pydantic schemas for API request/response validation and serialization.
# ============================================================================
import json # For parsing metadata JSON strings from Form
from datetime import datetime
import uuid # For UUID types implicitly
from typing import List, Dict, Any, Optional, Union
from pydantic import BaseModel, Field, HttpUrl, UUID4 # UUID4, HttpUrl from Pydantic
from enum import Enum # For enums


# --- General Schemas ---
class ErrorCode(str, Enum):
    PARAMETER_INVALID = "PARAMETER_INVALID"
    PARAMETER_MISSING = "PARAMETER_MISSING"
    RESOURCE_NOT_FOUND = "RESOURCE_NOT_FOUND"
    UNAUTHORIZED = "UNAUTHORIZED"
    FORBIDDEN = "FORBIDDEN"
    IDEMPOTENCY_KEY_CONFLICT = "IDEMPOTENCY_KEY_CONFLICT"
    PAYLOAD_TOO_LARGE = "PAYLOAD_TOO_LARGE"
    UNSUPPORTED_FILE_TYPE = "UNSUPPORTED_FILE_TYPE"
    ANALYSIS_FAILED = "ANALYSIS_FAILED"
    INVALID_URL = "INVALID_URL"
    ENTITY_ALREADY_EXISTS = "ENTITY_ALREADY_EXISTS"
    INTERNAL_SERVER_ERROR = "INTERNAL_SERVER_ERROR"

class ErrorMessage(BaseModel):
    """Schema for a single error detail."""
    code: ErrorCode = Field(..., example="RESOURCE_NOT_FOUND", description="A machine-readable error code.")
    message: str = Field(..., example="File content with ID 'invalid-id' not found.", description="A human-readable description of the error.")
    parameter: Optional[str] = Field(None, example="file_url", description="The specific parameter that caused the error (if applicable).")

class ErrorResponse(BaseModel):
    """Standard error response wrapper, containing a list of error messages."""
    errors: List[ErrorMessage]

    model_config = {
        "json_schema_extra": {
            "examples": [
                {"errors": [{"code": "PARAMETER_INVALID", "message": "Invalid file_type provided.", "parameter": "file_type"}]}
            ]
        }
    }


class AsyncResponse(BaseModel):
    """Schema for asynchronous operation initiation responses (202 Accepted)."""
    operation_id: UUID4 = Field(..., example="e1f2b3a4-c5d6-7890-abcd-1234567890ef", description="Unique ID for the asynchronous operation being processed.")
    resource_id: Optional[UUID4] = Field(None, example="f8c0062b-92f7-4180-87a4-e9185a5323a6", description="The ID of the primary resource created or affected by this operation.")
    status: str = Field(..., example="queued", description="The current high-level status of the asynchronous operation's immediate acceptance.")
    message: Optional[str] = Field(None, example="File content received and queued for ingestion and analysis.", description="A descriptive message about the asynchronous operation.")

# --- FileContent Schemas ---
class FileContentStatus(str, Enum):
    UPLOADED = "uploaded"
    INGESTION_FAILED = "ingestion_failed"
    ANALYSIS_PENDING = "analysis_pending"
    ANALYSIS_COMPLETE = "analysis_complete"

class FileContentBase(BaseModel):
    name: str = Field(..., min_length=1, max_length=255, example="Project_Report_Q4.pdf")
    file_type: str = Field(..., min_length=1, max_length=100, example="application/pdf")
    external_id: Optional[str] = Field(None, max_length=180, example="CRM_ID_54321")
    analysis_profile_id: Optional[UUID4] = Field(None, example="d4e5f6a7-b8c9-0d12-e3f4-567890abcdef")
    initial_metadata: Optional[Dict[str, str]] = Field(None, example={"author": "Alex Kim", "project": "Gamma Initiative"})

class FileContentCreateRequestMultipart(FileContentBase):
    """Request schema for file upload via multipart/form-data."""
    # file_bytes is handled directly by FastAPI's UploadFile
    pass

class FileContentCreateRequestUrl(FileContentBase):
    """Request schema for file upload via URL."""
    file_url: HttpUrl = Field(..., example="https://my-cloud-storage.com/files/document_v2.docx")

class FileContent(FileContentBase):
    id: UUID4 = Field(..., example="a1b2c3d4-e5f6-7890-1234-567890abcdef")
    object: str = Field("file_content", const=True)
    created_at: datetime
    updated_at: datetime
    size_bytes: int = Field(..., example=1245678)
    status: FileContentStatus = Field(..., example=FileContentStatus.ANALYSIS_COMPLETE)
    storage_path: str = Field(..., example="/internal/storage/bucket/a1b2c3d4.pdf") # Internal detail
    semantic_ledger_account_id: Optional[UUID4] = Field(None, example="b2c3d4e5-f6a7-8901-2345-67890abcdeff")
    latest_analysis_job_id: Optional[UUID4] = Field(None, example="c3d4e5f6-a7b8-9012-3456-7890abcdef12")

    model_config = {
        'from_attributes': True,
        'json_schema_extra': {
            'examples': [
                {
                    "id": "f1234567-8901-2345-6789-0123456789ab",
                    "name": "Annual_Shareholder_Meeting_2023.pdf",
                    "file_type": "application/pdf",
                    "external_id": "SHRHLD_DOC_001",
                    "analysis_profile_id": None,
                    "initial_metadata": {"department": "Investor Relations"},
                    "object": "file_content",
                    "created_at": "2023-10-20T08:00:00Z",
                    "updated_at": "2023-10-20T08:00:00Z",
                    "size_bytes": 500123,
                    "status": "uploaded",
                    "storage_path": "./local_file_storage/f1234567-8901-2345-6789-0123456789ab.pdf",
                    "semantic_ledger_account_id": "a2b3c4d5-e6f7-8901-2345-67890abcdeff",
                    "latest_analysis_job_id": None
                }
            ]
        }
    }


# --- AnalysisProfile Schemas ---
class AnalysisStepType(str, Enum):
    TEXT_KEYWORD_EXTRACTION = "text_keyword_extraction"
    TEXT_ENTITY_RECOGNITION = "text_entity_recognition"
    TEXT_SENTIMENT_ANALYSIS = "text_sentiment_analysis"
    TEXT_SUMMARIZATION = "text_summarization"
    TEXT_CLASSIFICATION = "text_classification"
    TEXT_VECTOR_EMBEDDING = "text_vector_embedding"
    IMAGE_OBJECT_DETECTION = "image_object_detection"
    IMAGE_FACE_RECOGNITION = "image_face_recognition"
    IMAGE_SCENE_DESCRIPTION = "image_scene_description"
    AUDIO_TRANSCRIPTION = "audio_transcription"
    AUDIO_SPEAKER_DIARIZATION = "audio_speaker_diarization"
    DOCUMENT_STRUCTURE_PARSING = "document_structure_parsing"
    CUSTOM_SCRIPT = "custom_script"


class AnalysisStep(BaseModel):
    step_type: AnalysisStepType = Field(..., example=AnalysisStepType.TEXT_ENTITY_RECOGNITION,
                            description="The specific type of AI analysis to be performed in this step.")
    model_identifier: Optional[str] = Field(None, example="huggingface-distilbert-ner-v2.1",
                                           description="Identifier for a specific AI model version used.")
    parameters: Optional[Dict[str, Any]] = Field(None, example={"labels_to_extract": ["PERSON", "ORG"]},
                                                description="Customizable parameters for the analysis step, varies by `step_type`.")
    description: Optional[str] = Field(None, example="Identify named entities using a pre-trained model.")

class AnalysisProfileStatus(str, Enum):
    ACTIVE = "active"
    ARCHIVED = "archived"

class AnalysisProfileBase(BaseModel):
    name: str = Field(..., min_length=1, max_length=255, example="Legal Contract Review: Clauses and Parties")
    description: Optional[str] = Field(None, example="Profile optimized for extracting legal clauses from contracts.")
    target_file_types: List[str] = Field(..., min_items=1, example=["application/pdf"])
    analysis_steps: List[AnalysisStep] = Field(..., min_items=1, description="Ordered list of AI processing steps to apply.")
    metadata: Optional[Dict[str, str]] = Field(None, example={"department_owner": "Legal Tech"})

class AnalysisProfileCreateRequest(AnalysisProfileBase):
    pass

class AnalysisProfile(AnalysisProfileBase):
    id: UUID4 = Field(..., example="d4e5f6a7-b8c9-0d12-e3f4-567890abcdef")
    object: str = Field("analysis_profile", const=True)
    created_at: datetime
    updated_at: datetime
    status: AnalysisProfileStatus = Field(..., example=AnalysisProfileStatus.ACTIVE)
    version: int = Field(..., example=1)

    model_config = {'from_attributes': True}


# --- AnalysisJob Schemas ---
class AnalysisJobStatus(str, Enum):
    QUEUED = "queued"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"

class AnalysisJobCreateRequest(BaseModel):
    file_content_id: UUID4 = Field(..., example="a1b2c3d4-e5f6-7890-1234-567890abcdef")
    analysis_profile_id: UUID4 = Field(..., example="d4e5f6a7-b8c9-0d12-e3f4-567890abcdef")
    metadata: Optional[Dict[str, str]] = Field(None, example={"triggered_by": "API user"})

class AnalysisJob(BaseModel):
    id: UUID4 = Field(..., example="c3d4e5f6-a7b8-9012-3456-7890abcdef12")
    object: str = Field("analysis_job", const=True)
    created_at: datetime
    updated_at: datetime
    file_content_id: UUID4
    analysis_profile_id: UUID4
    status: AnalysisJobStatus = Field(..., example=AnalysisJobStatus.COMPLETED)
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
    error_details: Optional[Dict[str, Any]] = Field(None, example={"code": "AI_INFERENCE_FAILURE", "message": "Model error."})
    semantic_ledger_transaction_id: Optional[UUID4] = Field(None, example="e1f2b3a4-c5d6-7890-abcd-1234567890ef")
    metadata: Optional[Dict[str, str]] = None

    model_config = {'from_attributes': True}

# --- Semantic Ledger Schemas ---
class SemanticDataType(str, Enum):
    KEYWORD = "keyword"
    TOPIC = "topic"
    ENTITY = "entity"
    SENTIMENT_SCORE = "sentiment_score"
    SUMMARY_EXTRACT = "summary_extract"
    VECTOR_EMBEDDING_HASH = "vector_embedding_hash" # For vector db reference
    TAXONOMY_NODE = "taxonomy_node"
    TEXT_EXCERPT = "text_excerpt"
    OBJECT_DETECTION = "object_detection"
    FACE_DETECTION = "face_detection"
    LANGUAGE = "language"
    COLOR_PALETTE = "color_palette"
    AUDIO_TRANSCRIPTION = "audio_transcription"
    DOCUMENT_STRUCTURE = "document_structure"

class SemanticData(BaseModel):
    type: SemanticDataType = Field(..., example=SemanticDataType.ENTITY, description="The specific category of semantic data extracted.")
    value: Union[str, int, float, bool, Dict[str, Any], List[Any]] = Field(..., example="Artificial Intelligence", description="The actual data value extracted. Can be complex.")
    confidence: Optional[float] = Field(None, ge=0.0, le=1.0, example=0.95)
    source_model_identifier: Optional[str] = Field(None, example="SpaCy_Large_NER_v3")
    original_file_excerpt: Optional[str] = Field(None, example="...breakthroughs in **Artificial Intelligence** were announced...")
    location_in_file: Optional[Dict[str, Any]] = Field(None, example={"page_number": 3, "char_offset_start": 501})
    classification_system: Optional[str] = Field(None, example="NAICS")
    concept_external_id: Optional[str] = Field(None, example="CONCEPT::Technology_Innovations", description="External ID of associated SemanticConceptAccount.")
    timestamp_in_file: Optional[float] = Field(None, example=65.23, description="For time-based media, timestamp in seconds.")


class SemanticLedgerAccountType(str, Enum):
    FILE_CONTENT_ACCOUNT = "file_content_account"
    SEMANTIC_CONCEPT_ACCOUNT = "semantic_concept_account"

class SemanticLedgerAccountBase(BaseModel):
    name: str = Field(..., min_length=1, max_length=255, example="Topic: Quantum Computing")
    description: Optional[str] = Field(None, example="Central account for all content related to Quantum Computing.")
    metadata: Optional[Dict[str, str]] = Field(None, example={"department_owner": "Data Science"})

class SemanticLedgerAccountCreateRequest(SemanticLedgerAccountBase):
    external_id: str = Field(..., min_length=1, max_length=255, example="CONCEPT::Quantum_Computing")
    type: SemanticLedgerAccountType = Field(SemanticLedgerAccountType.SEMANTIC_CONCEPT_ACCOUNT, const=True)

class SemanticLedgerAccountUpdateRequest(BaseModel):
    name: Optional[str] = Field(None, min_length=1, max_length=255)
    description: Optional[str] = None
    metadata: Optional[Dict[str, str]] = Field(None)

class SemanticLedgerAccount(SemanticLedgerAccountBase):
    id: UUID4 = Field(..., example="a1b2c3d4-e5f6-7890-abcd-1234567890ef")
    object: str = Field("semantic_ledger_account", const=True)
    created_at: datetime
    updated_at: datetime
    external_id: str = Field(..., example="CONCEPT::Artificial_Intelligence_Ethics")
    type: SemanticLedgerAccountType = Field(..., example=SemanticLedgerAccountType.SEMANTIC_CONCEPT_ACCOUNT)
    related_file_content_id: Optional[UUID4] = Field(None, example="f8c0062b-92f7-4180-87a4-e9185a5323a6")
    latest_insight_transaction_id: Optional[UUID4] = Field(None, example="c3d4e5f6-a7b8-9012-3456-7890abcdef12")

    model_config = {'from_attributes': True}

class SemanticLedgerEntry(BaseModel):
    id: UUID4 = Field(..., example="a1b2c3d4-e5f6-7890-abcd-1234567890ef")
    object: str = Field("semantic_ledger_entry", const=True)
    created_at: datetime
    updated_at: datetime
    semantic_data: SemanticData
    file_content_id: UUID4 = Field(..., example="f8c0062b-92f7-4180-87a4-e9185a5323a6")
    ledger_account_id: UUID4 = Field(..., example="b2c3d4e5-f6a7-8901-2345-67890abcdeff")
    semantic_transaction_id: UUID4 = Field(..., example="c3d4e5f6-a7b8-9012-3456-7890abcdef12")
    metadata: Optional[Dict[str, str]] = Field(None)

    model_config = {'from_attributes': True}


class SemanticLedgerEntryCreateRequest(BaseModel):
    semantic_data: SemanticData
    file_content_id: UUID4
    ledger_account_id: UUID4
    semantic_transaction_id: UUID4 # Must provide, as it's part of the immutable transaction
    metadata: Optional[Dict[str, str]] = None

class SemanticLedgerTransaction(BaseModel):
    id: UUID4 = Field(..., example="a1b2c3d4-e5f6-7890-abcd-1234567890ef")
    object: str = Field("semantic_ledger_transaction", const=True)
    created_at: datetime
    updated_at: datetime
    description: Optional[str] = Field(None)
    status: str = Field("posted", const=True)
    external_id: str = Field(..., example="e1f2b3a4-c5d6-7890-abcd-1234567890ef") # AnalysisJob ID
    file_content_id: UUID4
    analysis_profile_id: UUID4
    number_of_semantic_entries: int
    processing_time_ms: Optional[int] = None
    metadata: Optional[Dict[str, str]] = Field(None)

    model_config = {'from_attributes': True}

class SemanticLedgerTransactionCreateRequest(BaseModel):
    description: Optional[str] = None
    external_id: str # This is usually the AnalysisJob ID
    file_content_id: UUID4
    analysis_profile_id: UUID4
    number_of_semantic_entries: int
    processing_time_ms: Optional[int] = None
    metadata: Optional[Dict[str, str]] = None


# --- Document Schemas ---
class DocumentableType(str, Enum):
    FILE_CONTENT = "file_content"
    ANALYSIS_JOB = "analysis_job"
    SEMANTIC_LEDGER_ACCOUNT = "semantic_ledger_account"

class DocumentUploadRequest(BaseModel):
    documentable_id: UUID4 = Field(..., description="The unique ID of the entity this document is associated with.")
    documentable_type: DocumentableType = Field(..., description="The type of the associated entity.")
    document_type: Optional[str] = Field(None, example="ai_audit_report")
    metadata: Optional[Dict[str, str]] = Field(None)

class Document(BaseModel):
    id: UUID4
    object: str = Field("document", const=True)
    created_at: datetime
    updated_at: datetime
    document_type: Optional[str] = None
    documentable_id: UUID4
    documentable_type: DocumentableType
    file_name: str
    content_type: str
    file_size_bytes: int
    storage_path: str
    metadata: Optional[Dict[str, str]] = None

    model_config = {'from_attributes': True}

# --- Event Schemas ---
class PlatformEventName(str, Enum):
    FILE_CONTENT_UPLOADED = "file_content.uploaded"
    FILE_CONTENT_INGESTION_FAILED = "file_content.ingestion_failed"
    ANALYSIS_JOB_QUEUED = "analysis_job.queued"
    ANALYSIS_JOB_PROCESSING = "analysis_job.processing"
    ANALYSIS_JOB_COMPLETED = "analysis_job.completed"
    ANALYSIS_JOB_FAILED = "analysis_job.failed"
    SEMANTIC_LEDGER_ENTRY_ADDED = "semantic_ledger_entry.added"
    SEMANTIC_LEDGER_CONCEPT_CREATED = "semantic_ledger_concept.created"

class PlatformEvent(BaseModel):
    id: UUID4
    object: str = Field("event", const=True)
    created_at: datetime
    event_name: PlatformEventName = Field(..., example=PlatformEventName.ANALYSIS_JOB_COMPLETED)
    event_time: datetime
    entity_id: Optional[UUID4] = None
    resource_type: Optional[str] = None
    data: Optional[Dict[str, Any]] = None

    model_config = {'from_attributes': True}


# ============================================================================
# FILE: nexusmind-ai/src/services/file_ingestion.py
# DESCRIPTION: Backend service for handling file storage operations.
# ============================================================================
import aiofiles # For asynchronous file operations
import os # For path manipulation and file system operations
# from src.config import settings # Config is global here

async def save_uploaded_file_to_local_storage(file_content_id: uuid.UUID, file_data: UploadFile, original_filename: str) -> str:
    """
    Saves an uploaded `UploadFile` (from FastAPI) to the configured local storage path.
    Constructs a unique filename using the `file_content_id` and original extension.
    Returns the full path where the file was saved.
    In production, this would typically integrate with cloud blob storage like S3 or GCS.
    """
    # Extract file extension safely
    _, ext = os.path.splitext(original_filename)
    # Construct a unique filename using the file_content_id to prevent clashes
    unique_filename = f"{file_content_id}{ext}"
    # Full path to save the file
    file_path = os.path.join(settings.FILE_STORAGE_PATH, unique_filename)

    # Asynchronously write the file content in chunks for efficiency
    try:
        async with aiofiles.open(file_path, "wb") as f:
            while content := await file_data.read(8192): # Read and write in 8KB chunks
                await f.write(content)
        return file_path
    except Exception as e:
        # Clean up any partial file in case of error
        if os.path.exists(file_path):
            os.remove(file_path)
        raise IOError(f"Failed to save file to local storage: {e}")

async def fetch_file_from_url_and_save(file_content_id: uuid.UUID, file_url: HttpUrl, original_filename: str) -> tuple[str, int]:
    """
    Simulates fetching a file from a URL and saving it to local storage.
    In a real system, this would use an HTTP client (e.g., aiohttp, httpx) to
    download the file, potentially with stream parsing and error handling for large files.
    Returns the path where saved and its size.
    """
    import httpx # Placeholder for real HTTP requests

    # Extract file extension safely
    _, ext = os.path.splitext(original_filename)
    unique_filename = f"{file_content_id}{ext}"
    file_path = os.path.join(settings.FILE_STORAGE_PATH, unique_filename)

    print(f"Simulating fetch from {file_url} and saving to {file_path}") # Log for simulation

    # In a real scenario, this would be an actual HTTP GET request and stream writing
    # For this recipe, we simulate by creating a dummy file.
    dummy_content = b"This is simulated file content fetched from a URL."
    try:
        async with aiofiles.open(file_path, "wb") as f:
            await f.write(dummy_content)
        simulated_size = len(dummy_content)
        return file_path, simulated_size
    except Exception as e:
        if os.path.exists(file_path):
            os.remove(file_path)
        raise IOError(f"Simulated file fetch failed: {e}")

async def delete_file_from_local_storage(file_path: str):
    """
    Deletes a file from the configured local storage path.
    In production, this would interact with the cloud blob storage API.
    """
    if os.path.exists(file_path):
        try:
            os.remove(file_path)
            print(f"Deleted file from storage: {file_path}")
        except Exception as e:
            # Log error if file deletion fails but don't prevent DB transaction from completing
            print(f"Warning: Failed to delete file {file_path} from storage: {e}")

# ============================================================================
# FILE: nexusmind-ai/src/services/ai_analysis.py
# DESCRIPTION: Backend service for simulating AI analysis.
# ============================================================================
# import uuid # for internal UUID generation within semantic data
# from datetime import datetime # for setting creation times within mock semantic entries
# from typing import List, Dict, Any, Optional # for type hints

async def run_simulated_ai_analysis(file_path: str, analysis_profile_config: Dict[str, Any]) -> List[Dict[str, Any]]:
    """
    Simulates running an AI analysis on a file and generates a list of
    raw semantic data insights. This is a placeholder for real AI/ML model inference.
    Returns a list of dictionaries matching the SemanticData schema structure.
    """
    print(f"Simulating AI analysis for file: {file_path} with profile: {analysis_profile_config['name']}")

    simulated_entries = []
    
    # Simulate based on step types in the profile configuration
    for step_config in analysis_profile_config['analysis_steps']:
        step_type = step_config.get("step_type")
        model_identifier = step_config.get("model_identifier", "simulated-ai-v1.0")
        
        # Example 1: Text Keyword Extraction
        if "text_keyword_extraction" in step_type:
            keywords = ["simulated_data", "nexusmind", "ai", "content_ledger"]
            for keyword in keywords:
                simulated_entries.append({
                    "type": "keyword",
                    "value": keyword,
                    "confidence": round(0.7 + (hash(keyword) % 100) / 1000.0, 2), # Some varying confidence
                    "source_model_identifier": model_identifier,
                    "original_file_excerpt": "Relevant text...",
                    "location_in_file": {"page_number": 1} # Dummy location
                })
        
        # Example 2: Text Sentiment Analysis
        if "text_sentiment_analysis" in step_type:
            simulated_entries.append({
                "type": "sentiment_score",
                "value": -0.35, # Example negative sentiment
                "confidence": 0.88,
                "source_model_identifier": model_identifier,
                "original_file_excerpt": "Text expressing concerns..."
            })
        
        # Example 3: Entity Recognition
        if "text_entity_recognition" in step_type:
            entities = [
                {"type": "PERSON", "name": "Dr. Evelyn Reed"},
                {"type": "ORGANIZATION", "name": "Global Research Inc."},
            ]
            for entity in entities:
                simulated_entries.append({
                    "type": "entity",
                    "value": entity['name'],
                    "confidence": 0.95,
                    "source_model_identifier": model_identifier,
                    "original_file_excerpt": f"Entity found: {entity['name']}",
                    "location_in_file": {},
                    "concept_external_id": f"ENTITY::{entity['name'].replace(' ', '_')}" # Link to potential concept account
                })

        # Example 4: Vector Embedding Hash (real vector would be stored in VectorDB)
        if "text_vector_embedding" in step_type:
             simulated_entries.append({
                "type": "vector_embedding_hash",
                "value": str(uuid.uuid4()), # Placeholder for a real vector's hash/ID
                "confidence": 1.0, # Embedding generation is usually deterministic
                "source_model_identifier": model_identifier
            })
        
        # Add more simulated logic for other step_types...
        # Image object detection:
        if "image_object_detection" in step_type:
             simulated_entries.append({
                "type": "object_detection",
                "value": {"label": "table", "bbox": {"x": 0.1, "y": 0.2, "width": 0.5, "height": 0.3}},
                "confidence": 0.85,
                "source_model_identifier": model_identifier,
                "original_file_excerpt": "Detected a table structure.",
                "location_in_file": {"bounding_box": {"x": 0.1, "y": 0.2, "width": 0.5, "height": 0.3}}
            })

    return simulated_entries

# ============================================================================
# FILE: nexusmind-ai/src/tasks/celery_tasks.py
# DESCRIPTION: Celery tasks for background processing of files and analysis.
# ============================================================================
import uuid # For creating UUIDs if needed in task logic
import os # For interacting with file system
from datetime import datetime
from celery import Celery # Core Celery import
from celery.signals import task_postrun # For handling task outcomes

# Re-import all needed dependencies for Celery workers
from src.config import settings
from src.database import AsyncSessionLocal # For async DB operations in tasks
from sqlalchemy.future import select
from src.models import (
    FileContent as DBFileContent,
    AnalysisJob as DBAnalysisJob,
    AnalysisProfile as DBAnalysisProfile,
    SemanticLedgerAccount as DBSemanticLedgerAccount,
    SemanticLedgerTransaction as DBSemanticLedgerTransaction,
    SemanticLedgerEntry as DBSemanticLedgerEntry
)
from src.schemas import (
    AnalysisJobStatus, FileContentStatus, SemanticLedgerTransactionCreateRequest,
    SemanticLedgerEntryCreateRequest, SemanticData, SemanticDataType,
    SemanticLedgerAccountType
) # For data validation within task
from src.services.file_ingestion import fetch_file_from_url_and_save
from src.services.ai_analysis import run_simulated_ai_analysis


# --- Celery Application Initialization ---
# This is the Celery worker entry point, configuring it with the broker and backend.
celery_app = Celery(
    'nexusmind_tasks',
    broker=settings.CELERY_BROKER_URL,
    backend=settings.CELERY_RESULT_BACKEND,
    include=['src.tasks.celery_tasks'] # Important for Celery to find tasks
)

# Optional: Celery task configuration
celery_app.conf.update(
    task_track_started=True, # Enable tracking task 'started' state
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    enable_utc=True,
    broker_connection_retry_on_startup=True # Retry connecting to broker on worker start
)

# --- Celery Task: Process File Ingestion and Analysis ---
@celery_app.task(bind=True, name="process_file_ingestion_and_analysis")
async def process_file_ingestion_and_analysis(
    self, # 'self' when bind=True gives access to task instance attributes like request.id
    file_content_id: str,
    analysis_profile_id: Optional[str],
    external_url: Optional[str]
):
    """
    Celery task to handle file ingestion (especially for URL-based), update
    FileContent status, trigger an AnalysisJob, perform (simulated) AI analysis,
    and post semantic entries to the ledger.
    """
    db_session: AsyncSession = AsyncSessionLocal() # Get a new async DB session for this task
    file_content_db = None
    analysis_job_db = None
    semantic_ledger_account_db = None

    try:
        # Step 1: Update AnalysisJob status to 'processing'
        # The job might be created here or pre-created by API if triggered directly
        # For simplicity, assume task.request.id is mapped to an AnalysisJob
        # If it's a direct file_content_id from upload endpoint, then an analysis job needs creating first.
        # Let's adjust: the upload endpoint already initiated AsyncResponse with task_result.id as operation_id.
        # We need to *create* the analysis job first in the task.
        
        # First, retrieve the FileContent and associated Semantic Ledger Account
        file_content_uuid = uuid.UUID(file_content_id)
        stmt_fc = select(DBFileContent).where(DBFileContent.id == file_content_uuid)
        result_fc = await db_session.execute(stmt_fc)
        file_content_db = result_fc.scalar_one_or_none()

        if not file_content_db:
            raise ValueError(f"FileContent {file_content_id} not found for analysis.")
            
        stmt_sla = select(DBSemanticLedgerAccount).where(DBSemanticLedgerAccount.related_file_content_id == file_content_uuid)
        result_sla = await db_session.execute(stmt_sla)
        semantic_ledger_account_db = result_sla.scalar_one_or_none()
        
        if not semantic_ledger_account_db:
            # This should ideally be created with the file, but as a fallback/check:
            # This needs to mirror `get_or_create_file_content_ledger_account` logic, or better yet, be guaranteed pre-creation.
            # For this consolidated script, assuming the main endpoint reliably creates it.
            # If not, would need to re-implement `get_or_create_file_content_ledger_account` inline here
            # Or make a transactional backend service for that part
            print(f"WARNING: Semantic Ledger Account for file {file_content_id} not found in task. This should be pre-created.")


        # Retrieve the analysis profile config if provided
        analysis_profile_db = None
        if analysis_profile_id:
            profile_uuid = uuid.UUID(analysis_profile_id)
            stmt_profile = select(DBAnalysisProfile).where(DBAnalysisProfile.id == profile_uuid)
            result_profile = await db_session.execute(stmt_profile)
            analysis_profile_db = result_profile.scalar_one_or_none()
            if not analysis_profile_db:
                raise ValueError(f"Analysis Profile {analysis_profile_id} not found.")

        # Create/Update Analysis Job based on whether it was already implicitly made (e.g. from FileContent POST with analysis_profile_id)
        # If process_file_ingestion_and_analysis task is always linked to one-to-one AnalysisJob, we create here.
        # This task ID is usually what would update this specific analysis_job.id.
        analysis_job_id = uuid.UUID(self.request.id) # Use Celery task ID as AnalysisJob ID

        analysis_job_db = DBAnalysisJob(
            id=analysis_job_id,
            file_content_id=file_content_db.id,
            analysis_profile_id=analysis_profile_db.id if analysis_profile_db else None,
            status=AnalysisJobStatus.PROCESSING.value, # Update status
            started_at=datetime.utcnow(),
            metadata_={"celery_task_id": str(self.request.id)},
            created_at=datetime.utcnow(), # Manually set, as BaseModelMixin is on class, not instance.
            updated_at=datetime.utcnow()
        )
        db_session.add(analysis_job_db)
        await db_session.commit()
        await db_session.refresh(analysis_job_db)


        # --- Ingestion Step (for external URLs or other pre-processing) ---
        if external_url:
            print(f"Fetching content for FileContent {file_content_db.id} from URL: {external_url}")
            try:
                # Replace with actual HTTP fetch logic from `file_ingestion.py`
                fetched_file_path, fetched_file_size = await fetch_file_from_url_and_save(
                    file_content_db.id,
                    external_url,
                    file_content_db.name
                )
                file_content_db.storage_path = fetched_file_path
                file_content_db.size_bytes = fetched_file_size
                await db_session.commit() # Update file_content_db with final path/size
                await db_session.refresh(file_content_db)
            except Exception as e:
                # Mark file content and job as failed if ingestion fails
                file_content_db.status = FileContentStatus.INGESTION_FAILED.value
                analysis_job_db.status = AnalysisJobStatus.FAILED.value
                analysis_job_db.error_details = {"code": ErrorCode.INVALID_URL.value, "message": f"URL ingestion failed: {str(e)}"}
                analysis_job_db.completed_at = datetime.utcnow()
                await db_session.commit()
                raise e # Re-raise to mark task as failed in Celery backend

        
        # Step 2: Perform AI analysis (simulated) if analysis_profile is linked
        semantic_entries_raw = []
        if analysis_profile_db:
            print(f"Starting simulated AI analysis for FileContent {file_content_db.id} using profile {analysis_profile_db.name}")
            try:
                processing_start_time = datetime.utcnow()
                semantic_entries_raw = await run_simulated_ai_analysis(file_content_db.storage_path, analysis_profile_db.model_dump(mode='json')) # Pass profile as dict
                processing_end_time = datetime.utcnow()
                processing_duration_ms = int((processing_end_time - processing_start_time).total_seconds() * 1000)
                
                print(f"Simulated analysis complete. Generated {len(semantic_entries_raw)} entries.")
            except Exception as e:
                # Mark job as failed if AI analysis fails
                analysis_job_db.status = AnalysisJobStatus.FAILED.value
                analysis_job_db.error_details = {"code": ErrorCode.ANALYSIS_FAILED.value, "message": f"AI analysis failed: {str(e)}"}
                analysis_job_db.completed_at = datetime.utcnow()
                file_content_db.status = FileContentStatus.ANALYSIS_PENDING.value # Analysis didn't complete
                await db_session.commit()
                raise e

            # Step 3: Record SemanticLedgerTransaction and SemanticLedgerEntries
            semantic_transaction_id = uuid.uuid4()
            number_of_entries = len(semantic_entries_raw)
            
            # Create the SemanticLedgerTransaction
            semantic_transaction_db = DBSemanticLedgerTransaction(
                id=semantic_transaction_id,
                description=f"AI analysis results for '{file_content_db.name}' using profile '{analysis_profile_db.name}'",
                external_id=str(analysis_job_db.id), # Link to AnalysisJob
                file_content_id=file_content_db.id,
                analysis_profile_id=analysis_profile_db.id,
                number_of_semantic_entries=number_of_entries,
                processing_time_ms=processing_duration_ms,
                metadata_={"analysis_run_id": str(semantic_transaction_id)}, # Add transaction specific metadata
                created_at=datetime.utcnow(),
                updated_at=datetime.utcnow(),
            )
            db_session.add(semantic_transaction_db)
            await db_session.flush() # Flush to get semantic_transaction_db.id

            # Update related AnalysisJob and FileContent
            analysis_job_db.semantic_ledger_transaction_id = semantic_transaction_db.id
            file_content_db.latest_analysis_job_id = analysis_job_db.id
            # Also update SemanticLedgerAccount's latest transaction
            if semantic_ledger_account_db:
                semantic_ledger_account_db.latest_insight_transaction_id = semantic_transaction_db.id

            # Process raw semantic entries into DB objects
            semantic_entries_db_objects = []
            for entry_data in semantic_entries_raw:
                # Convert raw dict to SemanticData Pydantic model for validation
                try:
                    semantic_data_model = SemanticData(**entry_data)
                except Exception as e:
                    print(f"ERROR: Failed to validate semantic data: {e} for entry: {entry_data}. Skipping.")
                    continue
                
                # Check/create concept accounts if `concept_external_id` is present
                if semantic_data_model.concept_external_id:
                    concept_id = await get_or_create_semantic_concept_account(
                        db_session,
                        semantic_data_model.concept_external_id,
                        f"Concept: {semantic_data_model.concept_external_id.split('::')[-1]}", # Derived name
                    )
                    # Add an entry that tags the Concept Account with the FileContent
                    semantic_entries_db_objects.append(DBSemanticLedgerEntry(
                        id=uuid.uuid4(), # System generated UUID
                        semantic_data={ # Simple semantic data for concept linkage entry
                            "type": "file_linked",
                            "value": str(file_content_db.id),
                            "related_file_content_name": file_content_db.name,
                            "original_semantic_data_type": semantic_data_model.type,
                            "original_semantic_data_value": str(semantic_data_model.value) # Convert to string to avoid complex JSON types in simple concept linking entries
                        },
                        file_content_id=file_content_db.id,
                        ledger_account_id=concept_id, # This entry is posted to the CONCEPT account
                        semantic_transaction_id=semantic_transaction_id,
                        metadata_={"linked_by_original_entry": str(file_content_db.id)},
                        created_at=datetime.utcnow(),
                        updated_at=datetime.utcnow()
                    ))
                
                # Create the primary semantic ledger entry
                semantic_entries_db_objects.append(DBSemanticLedgerEntry(
                    id=uuid.uuid4(), # System generated UUID
                    semantic_data=semantic_data_model.model_dump(mode='json'), # Store as JSONB
                    file_content_id=file_content_db.id,
                    ledger_account_id=semantic_ledger_account_db.id, # This entry is posted to the FILE_CONTENT account
                    semantic_transaction_id=semantic_transaction_id,
                    metadata_=entry_data.get("metadata", {}),
                    created_at=datetime.utcnow(),
                    updated_at=datetime.utcnow()
                ))
            db_session.add_all(semantic_entries_db_objects)
        
        # Step 4: Finalize statuses and commit
        analysis_job_db.completed_at = datetime.utcnow()
        analysis_job_db.status = AnalysisJobStatus.COMPLETED.value
        file_content_db.status = FileContentStatus.ANALYSIS_COMPLETE.value
        
        await db_session.commit()
        
    except Exception as e:
        print(f"Error during async file processing for {file_content_id}: {e}")
        # Mark job and file content as failed
        if file_content_db:
            file_content_db.status = FileContentStatus.INGESTION_FAILED.value # Or ANALYSIS_FAILED, depending on where error occurred
        if analysis_job_db:
            analysis_job_db.status = AnalysisJobStatus.FAILED.value
            analysis_job_db.completed_at = datetime.utcnow()
            analysis_job_db.error_details = {"code": ErrorCode.INTERNAL_SERVER_ERROR.value, "message": str(e)}
        await db_session.rollback() # Rollback all changes if an error occurs
        raise e # Re-raise exception so Celery marks task as failed

    finally:
        await db_session.close() # Ensure session is closed

# Utility for semantic concept accounts (could be its own service)
async def get_or_create_semantic_concept_account(db: AsyncSession, external_id: str, name: str) -> uuid.UUID:
    """
    Retrieves or creates a SemanticLedgerAccount of type 'semantic_concept_account'.
    """
    stmt = select(DBSemanticLedgerAccount).where(DBSemanticLedgerAccount.external_id == external_id)
    result = await db.execute(stmt)
    existing_account = result.scalar_one_or_none()

    if existing_account:
        return existing_account.id

    new_concept_account = DBSemanticLedgerAccount(
        id=uuid.uuid4(), # System generated UUID
        name=name,
        description=f"Auto-generated concept account for {name}.",
        external_id=external_id,
        type=SemanticLedgerAccountType.SEMANTIC_CONCEPT_ACCOUNT.value,
        created_at=datetime.utcnow(), # Explicitly setting defaults
        updated_at=datetime.utcnow()
    )
    db.add(new_concept_account)
    await db.flush() # Flush to get the ID, but don't commit yet (task manages commit)
    return new_concept_account.id

# Add Celery signal for clean session closing in case of unexpected errors
@task_postrun.connect
def close_dbsession_if_needed(sender=None, task_id=None, task=None, args=None, kwargs=None, **kw):
    """Signal handler to ensure session is closed even if task fails unexpectedly."""
    if hasattr(sender, '_AsyncSessionLocal_task_session'):
        delattr(sender, '_AsyncSessionLocal_task_session')
    
# NOTE: Worker script is needed to run celery worker separately:
# `nexusmind-ai/src/worker.py` (simply imports and runs the celery app)
# from src.tasks.celery_tasks import celery_app
# celery_app.worker_main(['worker', '--loglevel=info', '-P', 'gevent'])
# Call using: `celery -A src.tasks.celery_tasks worker --loglevel=info -P gevent`

# ============================================================================
# FILE: nexusmind-ai/src/api/v1/file_contents.py
# DESCRIPTION: FastAPI endpoints for FileContent resource operations.
# ============================================================================
# Core imports (FastAPI, SQLAlchemy, etc.)
from fastapi import APIRouter, Depends, status, HTTPException, UploadFile, Form, Query, Header
from fastapi.responses import JSONResponse
from fastapi.encoders import jsonable_encoder
from sqlalchemy import select, and_, exists
from sqlalchemy.ext.asyncio import AsyncSession
from typing import List, Dict, Union, Annotated, Optional
import os # For file paths, not dependencies
import shutil # For file deletion simulation
import uuid # For UUID generation
from datetime import datetime # For current timestamps
import json # For parsing JSON strings from Form fields

# Internal project imports (Pydantic schemas, database access, Celery tasks, models)
from src.schemas import (
    FileContent,
    FileContentCreateRequestMultipart,
    FileContentCreateRequestUrl, # Though handled implicitly via Forms/URL here, conceptually still useful
    AsyncResponse,
    ErrorMessage,
    ErrorResponse,
    AnalysisProfileStatus, # To validate profile is active
    FileContentStatus,
    ErrorCode # For custom error codes
)
from src.database import get_db
from src.models import (
    FileContent as DBFileContent,
    AnalysisProfile as DBAnalysisProfile,
    AnalysisJob as DBAnalysisJob,
    SemanticLedgerAccount as DBSemanticLedgerAccount,
    SemanticLedgerTransaction as DBSemanticLedgerTransaction # Needed for FK update
)
from src.config import settings # Global app settings
from src.tasks.celery_tasks import process_file_ingestion_and_analysis # The background task


# --- FastAPI Router Initialization ---
# This router defines all the endpoints specifically for `/api/file_contents`.
router = APIRouter(prefix="/file_contents", tags=["FileContent"])


# --- Utility Function: Get or Create Semantic Ledger Account ---
# This helper is called during FileContent creation to provision its dedicated
# "Content Account" in the semantic ledger. Ensures idempotency of SLA creation.
async def get_or_create_file_content_ledger_account(db: AsyncSession, file_content_id: uuid.UUID, file_name: str) -> DBSemanticLedgerAccount:
    """
    Retrieves an existing or creates a new SemanticLedgerAccount of type 'file_content_account'
    for the given FileContent ID. This ensures each FileContent has a dedicated ledger space.
    """
    account_external_id = str(file_content_id) # The external ID for the SLA is the FileContent's UUID
    stmt = select(DBSemanticLedgerAccount).where(DBSemanticLedgerAccount.external_id == account_external_id)
    result = await db.execute(stmt)
    existing_account = result.scalar_one_or_none()

    if existing_account:
        return existing_account

    # Create a new SemanticLedgerAccount
    new_account = DBSemanticLedgerAccount(
        id=uuid.uuid4(),
        name=f"Content Account: {file_name}",
        description=f"Dedicated semantic ledger account for file content '{file_name}' (ID: {file_content_id}).",
        external_id=account_external_id,
        type=schemas.SemanticLedgerAccountType.FILE_CONTENT_ACCOUNT.value,
        related_file_content_id=file_content_id,
        # Default created_at/updated_at are set by BaseModelMixin, explicit set here is for clarity
        created_at=datetime.utcnow(),
        updated_at=datetime.utcnow()
    )
    db.add(new_account)
    await db.flush() # Flush to get ID if needed immediately, but not commit
    return new_account


# --- Endpoint: List File Contents ---
@router.get(
    "/",
    response_model=List[FileContent],
    summary="List File Contents",
    description="Retrieve a paginated and filterable list of uploaded digital assets. Supports filtering by file name, MIME type, processing status, external IDs, and custom metadata.",
    responses={
        200: {"description": "Successful retrieval of file contents.", "model": List[FileContent]},
        400: {"model": ErrorResponse, "description": "Invalid query parameters."},
    },
)
async def list_file_contents(
    db: AsyncSession = Depends(get_db),
    # Pagination parameters
    after_cursor: Optional[UUID4] = Query(None, description="UUID of the last item from the previous page for cursor-based pagination."),
    per_page: int = Query(50, ge=1, le=100, description="Number of results per page. Maximum 100."),
    # Filtering parameters
    name_contains: Optional[str] = Query(None, description="Case-insensitive partial match on file name."),
    file_type: Optional[str] = Query(None, description="Exact match on the file's MIME type."),
    status: Optional[schemas.FileContentStatus] = Query(None, description="Filter by file content processing status."),
    external_id: Optional[str] = Query(None, description="Exact match on a custom external ID for the file content."),
    metadata: Dict[str, str] = Query(
        default_factory=dict, # Initialize as empty dict if no metadata params
        description="Filter by custom metadata key-value pairs (e.g., `?metadata[project]=Alpha`). Accepts multiple key-value pairs using deepObject style. Only exact matches on metadata key-value pairs are supported."
    ),
) -> List[FileContent]:
    """
    **Description:**
    This endpoint allows you to search and retrieve records of digital content uploaded to NexusMind AI.
    It provides extensive filtering capabilities to narrow down your search results based on various criteria.

    **Query Parameters for Filtering:**
    - `after_cursor` (UUID): Used for efficient cursor-based pagination. Provide the `id` of the last item retrieved from the previous page to get the next set of results.
    - `per_page` (Integer): Controls the number of `FileContent` records to return per page.
    - `name_contains` (String): Filters records where the `name` field partially matches the provided string, ignoring case.
    - `file_type` (String): Filters records by an exact match of their MIME type.
    - `status` (Enum: `uploaded`, `ingestion_failed`, `analysis_pending`, `analysis_complete`): Filters records by their current processing status.
    - `external_id` (String): Filters records by an exact match of their custom `external_id`.
    - `metadata` (DeepObject: `Dict[str, str]`): Filters records by custom metadata. For example, `?metadata[author]=JaneDoe` will match files where metadata contains `{"author": "JaneDoe"}`.

    **Returns:**
    A list of `FileContent` objects serialized according to the `FileContent` schema. The response headers
    (`X-After-Cursor`, `X-Per-Page`) will assist in manual pagination.
    """
    stmt = select(DBFileContent)

    # Apply filters dynamically
    if name_contains:
        stmt = stmt.where(DBFileContent.name.ilike(f"%{name_contains}%"))
    if file_type:
        stmt = stmt.where(DBFileContent.file_type == file_type)
    if status:
        # Pydantic's `schemas.FileContentStatus` Enum handles validation of `status` automatically
        stmt = stmt.where(DBFileContent.status == status.value)
    if external_id:
        stmt = stmt.where(DBFileContent.external_id == external_id)
    
    # Handle dynamic JSONB metadata filtering
    if metadata:
        # Iterates through each key-value pair in the metadata dictionary provided
        for key, value in metadata.items():
            # SQLAlchemy's JSONB column allows filtering on specific keys within the JSON object.
            # `.astext` casts the JSON value at `key` to text for string comparison.
            stmt = stmt.where(DBFileContent.metadata_[key].astext == value)

    # Apply cursor-based pagination
    if after_cursor:
        # Retrieve the `created_at` timestamp of the `after_cursor` item
        cursor_item_stmt = select(DBFileContent.created_at).where(DBFileContent.id == after_cursor)
        cursor_result = await db.execute(cursor_item_stmt)
        cursor_created_at = cursor_result.scalar_one_or_none()

        if cursor_created_at:
            # For consistent pagination with `desc` ordering, find items created *before* the cursor's timestamp.
            # To handle duplicates created at the same microsecond (rare but possible), also use ID as tie-breaker.
            stmt = stmt.where(
                and_(
                    DBFileContent.created_at < cursor_created_at,
                    # No ID tie-breaker needed for this simple cursor example, but often critical
                )
            )
        else:
            # If after_cursor points to a non-existent item, it's a bad request.
            raise HTTPException(
                status_code=400,
                detail=ErrorResponse(errors=[ErrorMessage(
                    code=schemas.ErrorCode.PARAMETER_INVALID,
                    message=f"Provided `after_cursor` ID '{after_cursor}' does not exist or refers to an invalid record.",
                    parameter="after_cursor"
                )]).model_dump()
            )
        
    # Always order by created_at descending (newest first) to ensure consistent pagination order
    stmt = stmt.order_by(DBFileContent.created_at.desc()).limit(per_page)
    
    result = await db.execute(stmt)
    file_contents_db = result.scalars().all()

    # Convert SQLAlchemy ORM models to Pydantic models for consistent API response.
    # Pydantic's `model_validate` (with `from_attributes=True` in `FileContent` schema) handles the `metadata_` alias.
    return [FileContent.model_validate(fc) for fc in file_contents_db]


# --- Endpoint: Upload File Content and Initiate Analysis ---
@router.post(
    "/",
    response_model=AsyncResponse,
    status_code=status.HTTP_202_ACCEPTED,
    summary="Upload File Content and Initiate Analysis",
    description="Accepts a digital file directly via multipart form data OR an external URL. NexusMind AI then stores it, provisions its dedicated Content Account in the semantic ledger, and queues it for asynchronous AI analysis.",
    responses={
        202: {"description": "File upload and analysis initiation accepted. Processing is asynchronous. Returns an `AsyncResponse` for task tracking."},
        400: {"model": ErrorResponse, "description": "Invalid input parameters (e.g., missing file, unsupported type, malformed metadata JSON)."},
        413: {"model": ErrorResponse, "description": "Payload too large, or file size exceeds configured limits."}, # Note: FastAPI handles body size via server config (e.g. uvicorn worker_class=...), this is a conceptual response.
        409: {"model": ErrorResponse, "description": "Conflict, typically due to a duplicate `external_id`."},
    },
)
async def upload_file_content(
    db: AsyncSession = Depends(get_db),
    # FastAPI automatically handles multipart form data parsing into individual fields
    # Use Annotated[..., Form(...)] for form fields, and UploadFile for file parts.
    name: Annotated[str, Form(min_length=1, max_length=255, description="The human-readable name of the file (e.g., its original filename).")],
    file_type: Annotated[str, Form(min_length=1, max_length=100, description="The MIME type of the file (e.g., `application/pdf`, `image/jpeg`, `text/plain`).")],
    external_id: Annotated[Optional[str], Form(None, max_length=180, description="An optional unique identifier from your external system to link this FileContent.")] = None,
    analysis_profile_id: Annotated[Optional[UUID4], Form(None, description="The UUID of an existing `AnalysisProfile` to apply immediately after ingestion. If omitted, the file will be ingested but not analyzed.")] = None,
    initial_metadata: Annotated[Optional[str], Form(None, description="Optional custom key-value metadata for the file, provided as a JSON string (e.g., `'{\"author\":\"Jane Doe\",\"category\":\"Research\"}'`).")] = None,
    file_bytes: Annotated[Optional[UploadFile], Form(None, description="The binary content of the file. Required if `file_url` is not provided. Max upload size is server-dependent (e.g., configured in load balancer/reverse proxy).")] = None,
    file_url: Annotated[Optional[str], Form(None, description="A public or accessible URL from which NexusMind AI can download the file content. Required if `file_bytes` is not provided.")] = None,
    idempotency_key: Annotated[Optional[str], Header(alias="Idempotency-Key", description="A unique key (UUID, string) to ensure idempotency. If a request with the same key is re-sent, the previous success response will be returned without re-processing.")] = None # Not actively used in logic to return previous response for idempotency for brevity.
) -> AsyncResponse:
    """
    **Description:**
    This endpoint allows the creation of a new `FileContent` record in NexusMind AI, either by directly uploading
    the file bytes or by providing an external URL for NexusMind AI to fetch the content.

    Upon successful receipt of the file (or URL):
    1.  A `FileContent` record is immediately created in the database.
    2.  A dedicated `SemanticLedgerAccount` (known as a "Content Account") is provisioned for this file.
        All AI-derived insights from this file will be immutably logged into this specific ledger account.
    3.  An asynchronous background task is queued to handle:
        *   Fetching the file (if `file_url` was provided).
        *   Processing the file.
        *   If `analysis_profile_id` is specified, initiating the AI analysis based on that profile.
        *   Storing the resulting semantic entries in the ledger.

    **Behavior with `Idempotency-Key`:**
    If you send the same `Idempotency-Key` for identical request parameters and a successful response was already
    returned, NexusMind AI will return the original `202 Accepted` response, without re-processing the file.
    For this basic implementation, `idempotency_key` is collected but not fully implemented to replay previous results.

    **Error Handling:**
    -   A `400 Bad Request` is returned if both `file_bytes` and `file_url` are provided, or if neither is provided.
    -   `analysis_profile_id` is validated for existence and `file_type` compatibility before queuing the task.
    -   `initial_metadata` must be a valid JSON string convertible to `Dict[str, str]`.
    """
    # 1. Validate input combinations
    if not file_bytes and not file_url:
        raise HTTPException(
            status_code=400,
            detail=ErrorResponse(errors=[ErrorMessage(
                code=schemas.ErrorCode.PARAMETER_MISSING,
                message="Either 'file_bytes' (multipart/form-data) or 'file_url' (application/json or form field) must be provided for content ingestion.",
                parameter="file_bytes | file_url"
            )]).model_dump()
        )
    if file_bytes and file_url:
        raise HTTPException(
            status_code=400,
            detail=ErrorResponse(errors=[ErrorMessage(
                code=schemas.ErrorCode.PARAMETER_INVALID,
                message="Only one content source ('file_bytes' or 'file_url') can be provided. Please choose one method for file ingestion.",
                parameter="file_bytes & file_url"
            )]).model_dump()
        )

    # 2. Validate and Parse `initial_metadata` if present
    parsed_metadata: Optional[Dict[str, str]] = None
    if initial_metadata:
        try:
            parsed_metadata = json.loads(initial_metadata)
            if not isinstance(parsed_metadata, dict) or not all(isinstance(v, str) for v in parsed_metadata.values()):
                raise ValueError("Metadata must be a dictionary with string values.")
        except json.JSONDecodeError:
            raise HTTPException(
                status_code=400,
                detail=ErrorResponse(errors=[ErrorMessage(
                    code=schemas.ErrorCode.PARAMETER_INVALID,
                    message="`initial_metadata` must be a valid JSON string representing a dictionary with string values.",
                    parameter="initial_metadata"
                )]).model_dump()
            )
        except ValueError as e:
            raise HTTPException(
                status_code=400,
                detail=ErrorResponse(errors=[ErrorMessage(
                    code=schemas.ErrorCode.PARAMETER_INVALID,
                    message=f"`initial_metadata` invalid: {e}.",
                    parameter="initial_metadata"
                )]).model_dump()
            )

    # 3. Pre-validate Analysis Profile and File Type Compatibility
    analysis_profile_config: Optional[DBAnalysisProfile] = None
    if analysis_profile_id:
        profile_stmt = select(DBAnalysisProfile).where(
            and_(
                DBAnalysisProfile.id == analysis_profile_id,
                DBAnalysisProfile.status == schemas.AnalysisProfileStatus.ACTIVE.value # Only allow active profiles
            )
        )
        result_profile = await db.execute(profile_stmt)
        analysis_profile_config = result_profile.scalar_one_or_none()

        if not analysis_profile_config:
            raise HTTPException(
                status_code=400,
                detail=ErrorResponse(errors=[ErrorMessage(
                    code=schemas.ErrorCode.RESOURCE_NOT_FOUND,
                    message=f"Analysis Profile with ID '{analysis_profile_id}' not found or is not active. Please provide a valid active profile ID.",
                    parameter="analysis_profile_id"
                )]).model_dump()
            )
        
        # Check if file_type is supported by the chosen analysis_profile
        if file_type not in analysis_profile_config.target_file_types:
            raise HTTPException(
                status_code=400,
                detail=ErrorResponse(errors=[ErrorMessage(
                    code=schemas.ErrorCode.UNSUPPORTED_FILE_TYPE,
                    message=f"File type '{file_type}' is not supported by Analysis Profile '{analysis_profile_config.name}'. Supported types: {analysis_profile_config.target_file_types}.",
                    parameter="file_type"
                )]).model_dump()
            )
    
    # 4. Generate unique IDs and prepare storage path
    file_content_id = uuid.uuid4()
    # Safely get file extension or default to no extension if not present in original name
    file_extension = os.path.splitext(name)[1] 
    storage_filename = f"{file_content_id}{file_extension}"
    storage_full_path = os.path.join(settings.FILE_STORAGE_PATH, storage_filename)

    # Initialize new FileContent DB object
    new_file_content_db = DBFileContent(
        id=file_content_id,
        name=name,
        file_type=file_type,
        size_bytes=0, # Will be updated by async task for URL or here for bytes
        external_id=external_id,
        status=schemas.FileContentStatus.UPLOADED.value if file_bytes else schemas.FileContentStatus.ANALYSIS_PENDING.value,
        storage_path=storage_full_path,
        metadata_=parsed_metadata if parsed_metadata is not None else {}, # Ensure JSON column gets a dict
        # created_at and updated_at handled by BaseModelMixin
    )

    async with db.begin(): # Start a database transaction for atomicity
        # Step A: Provision a dedicated SemanticLedgerAccount (Content Account) for the new FileContent
        # This account will serve as the immutable ledger for all insights derived from this file.
        ledger_account = await get_or_create_file_content_ledger_account(db, new_file_content_db.id, new_file_content_db.name)
        new_file_content_db.semantic_ledger_account_id = ledger_account.id
        
        # Add the FileContent record to the database. Flush to generate its ID.
        db.add(new_file_content_db)
        await db.flush() 

        # Step B: If direct file bytes, save the file content to local storage.
        # This step occurs synchronously for multipart uploads for immediate feedback,
        # but the heavy AI analysis is still async.
        if file_bytes:
            # File saving logic from file_ingestion service
            try:
                # This call directly handles saving to disk
                file_disk_path = await services.file_ingestion.save_uploaded_file_to_local_storage(
                    new_file_content_db.id, file_bytes, name
                )
                new_file_content_db.storage_path = file_disk_path
                new_file_content_db.size_bytes = file_bytes.size # Use UploadFile's reported size
                new_file_content_db.status = schemas.FileContentStatus.UPLOADED.value
            except IOError as e:
                # If disk save fails, mark file content as failed ingestion immediately
                new_file_content_db.status = schemas.FileContentStatus.INGESTION_FAILED.value
                # No Celery task if immediate save failed.
                await db.rollback() # Rollback the DB changes including FileContent creation
                raise HTTPException(
                    status_code=500,
                    detail=ErrorResponse(errors=[ErrorMessage(
                        code=schemas.ErrorCode.INTERNAL_SERVER_ERROR,
                        message=f"Failed to save file bytes to storage: {str(e)}",
                        parameter="file_bytes"
                    )]).model_dump()
                )

        # Ensure the transaction is committed before queuing the Celery task.
        # The task will then pick up a fully persisted and valid FileContent record.
        await db.commit() 
    
    # 5. Queue asynchronous background task for full ingestion (if URL-based) and AI analysis
    # The `process_file_ingestion_and_analysis` Celery task handles the actual AI work.
    # We pass the ID of the newly created FileContent and the selected Analysis Profile ID.
    task_kwargs = {
        "file_content_id": str(new_file_content_db.id),
        "analysis_profile_id": str(analysis_profile_id) if analysis_profile_id else None,
        "external_url": str(file_url) if file_url else None,
    }
    
    # Send the task to Celery. The task's UUID (task_result.id) becomes our operation_id.
    task_result = process_file_ingestion_and_analysis.delay(**task_kwargs)

    # 6. Return asynchronous response
    return AsyncResponse(
        operation_id=uuid.UUID(task_result.id), # Use Celery task ID as our tracking operation_id
        resource_id=new_file_content_db.id,
        status="queued",
        message=f"File content '{new_file_content_db.name}' accepted. Ingestion and analysis queued (Operation ID: {task_result.id})."
    )


# --- Endpoint: Get File Content Details ---
@router.get(
    "/{id}",
    response_model=FileContent,
    summary="Get File Content Details",
    description="Retrieve detailed metadata and current processing status for a specific FileContent record by its unique ID. This endpoint provides insight into the file's lifecycle within NexusMind AI.",
    responses={
        200: {"description": "Successful retrieval of file content details."},
        404: {"model": ErrorResponse, "description": "File content with the specified ID was not found."},
    },
)
async def get_file_content(
    id: UUID4, # FastAPI automatically parses path UUID into uuid.UUID type
    db: AsyncSession = Depends(get_db)
) -> FileContent:
    """
    **Description:**
    Fetches the complete record of a single `FileContent` by its system-generated UUID.
    The response includes all associated metadata, its current processing `status`, the `storage_path`,
    and links to its dedicated `SemanticLedgerAccount` (Content Account) and the latest `AnalysisJob` ID.

    **Path Parameters:**
    - `id` (UUID): The unique system-generated identifier for the `FileContent` record to retrieve.

    **Returns:**
    A `FileContent` object, serialized according to its schema, representing the requested digital asset.

    **Error Conditions:**
    -   `404 Not Found`: If no `FileContent` record exists for the provided `id`.
    """
    stmt = select(DBFileContent).where(DBFileContent.id == id)
    result = await db.execute(stmt)
    file_content_db = result.scalar_one_or_none() # Retrieves a single ORM object or None

    if not file_content_db:
        # Raise HTTPException for 404 Not Found response
        raise HTTPException(
            status_code=404,
            detail=ErrorResponse(errors=[ErrorMessage(
                code=schemas.ErrorCode.RESOURCE_NOT_FOUND,
                message=f"File content with ID '{id}' not found in NexusMind AI.",
                parameter="id"
            )]).model_dump()
        )
    # Convert SQLAlchemy ORM model instance to Pydantic schema for API response
    return FileContent.model_validate(file_content_db)


# --- Endpoint: Delete File Content ---
@router.delete(
    "/{id}",
    status_code=status.HTTP_204_NO_CONTENT, # Standard response for successful deletion
    summary="Delete File Content",
    description="Permanently remove a `FileContent` record, its underlying file bytes, and all associated semantic data (semantic ledger entries, transactions, and analysis jobs) from NexusMind AI. This operation is irreversible and initiates a cascading cleanup.",
    responses={
        204: {"description": "File content and all associated data successfully deleted. No content is returned on success."},
        403: {"model": ErrorResponse, "description": "Forbidden. This could be due to insufficient permissions or specific internal policies preventing deletion."},
        404: {"model": ErrorResponse, "description": "File content with the specified ID was not found."},
        500: {"model": ErrorResponse, "description": "An unexpected internal server error occurred during deletion (e.g., storage cleanup failure)."},
    },
)
async def delete_file_content(
    id: UUID4,
    db: AsyncSession = Depends(get_db)
):
    """
    **Description:**
    Initiates the permanent deletion of a `FileContent` record identified by its UUID.
    This operation performs a cascade delete:
    1.  The `FileContent` record itself is removed from the database.
    2.  The physical file bytes are removed from NexusMind AI's storage (simulated as local disk cleanup).
    3.  All related `AnalysisJob` records for this `FileContent` are deleted (via `ondelete='CASCADE'` foreign key in `AnalysisJob` model).
    4.  The dedicated `SemanticLedgerAccount` (Content Account) for this file is deleted (via `ondelete='CASCADE'` in `SemanticLedgerAccount`).
    5.  All `SemanticLedgerEntry` objects associated with this file or its ledger account are deleted (via `ondelete='CASCADE'` foreign keys).
    6.  All `SemanticLedgerTransaction` objects generated from analyses of this file are deleted (via `ondelete='CASCADE'` foreign key).

    **WARNING:** This is a destructive and irreversible operation. All associated semantic insights derived
    from this file will also be permanently removed from your Intelli-Content Ledger.

    **Path Parameters:**
    - `id` (UUID): The unique identifier of the `FileContent` record to be deleted.

    **Returns:**
    An HTTP `204 No Content` status upon successful deletion.
    """
    stmt = select(DBFileContent).where(DBFileContent.id == id)
    result = await db.execute(stmt)
    file_content_db = result.scalar_one_or_none()

    if not file_content_db:
        # If the file content doesn't exist, return a 404
        raise HTTPException(
            status_code=404,
            detail=ErrorResponse(errors=[ErrorMessage(
                code=schemas.ErrorCode.RESOURCE_NOT_FOUND,
                message=f"File content with ID '{id}' not found in NexusMind AI.",
                parameter="id"
            )]).model_dump()
        )

    try:
        # Start a transaction to ensure atomicity of DB deletion
        async with db.begin_nested(): # Using nested for conceptual illustration, or direct 'await db.begin()'
            # First, attempt to delete the physical file from storage.
            # This is crucial for cleanup and storage management.
            await services.file_ingestion.delete_file_from_local_storage(file_content_db.storage_path)
            
            # Then, delete the FileContent record from the database.
            # Due to the CASCADE/SET NULL foreign key constraints defined in `src/models.py`,
            # this single operation in the DB will trigger automatic cleanup of linked records.
            # e.g., AnalysisJobs and SemanticLedgerAccounts explicitly linked by CASCADE on FileContent ID.
            # SemanticLedgerEntries and SemanticLedgerTransactions are often further cascaded from those.
            await db.delete(file_content_db)
            
            # The transaction will automatically commit or roll back via `db.begin_nested()` context.
        await db.commit() # Ensures outermost transaction commits.
            
    except Exception as e:
        # Rollback DB changes if any error occurs (e.g., storage deletion fails, although `delete_file_from_local_storage` already has internal try-except)
        # Note: If `begin_nested` is used for atomicity within one transaction, this will ensure rollback if needed
        # In this context (single top-level async with db.begin()), direct rollback for `e`
        print(f"ERROR: Failed to delete file content {id} and its associated data: {e}")
        # Raising HTTPException for API error response
        raise HTTPException(
            status_code=500,
            detail=ErrorResponse(errors=[ErrorMessage(
                code=schemas.ErrorCode.INTERNAL_SERVER_ERROR,
                message=f"An internal error occurred while deleting file content with ID '{id}'. Please try again. Details: {str(e)}",
            )]).model_dump()
        )

    # Return 204 No Content for successful deletion
    return JSONResponse(status_code=status.HTTP_204_NO_CONTENT)


# ============================================================================
# FILE: nexusmind-ai/src/api/routers.py
# DESCRIPTION: Aggregates all API routers under the /api prefix.
# ============================================================================
from fastapi import APIRouter
from src.api.v1 import file_contents # Import the FileContent router
import datetime # for timestamp in health check


api_router = APIRouter()

# --- API Health Check Endpoint ---
# This basic endpoint confirms the API service is running.
@api_router.get("/health", summary="API Health Check")
async def health_check():
    """
    Returns 'OK' if the NexusMind AI API server is running and responsive.
    Useful for quick operational status checks.
    """
    return {"status": "OK", "timestamp": datetime.datetime.utcnow().isoformat() + "Z"}

# --- Include Versioned Routers ---
# Mounts the FileContent router under the /api prefix.
api_router.include_router(file_contents.router) # No additional prefix needed here as `file_contents.router` already has one (`/file_contents`)


# ============================================================================
# FILE: nexusmind-ai/src/main.py
# DESCRIPTION: Main FastAPI application entry point.
# ============================================================================
from fastapi import FastAPI
from fastapi.responses import RedirectResponse
# from src.config import settings # Assuming config is globally loaded
# from src.api.routers import api_router # Assuming routers are defined


# --- FastAPI App Initialization ---
# Creates the main FastAPI application instance with global metadata and documentation paths.
app = FastAPI(
    title="NexusMind AI: Intelli-Content Ledger Platform API",
    description="API for AI-driven semantic content analysis and immutable knowledge provenance.",
    version="v1",
    openapi_url="/openapi.json", # Standard path for OpenAPI specification
    docs_url="/docs",            # Path for Swagger UI documentation
    redoc_url="/redoc",          # Path for ReDoc documentation
)

# --- Include Main API Router ---
# Mounts the top-level API router (`api_router`) under the `/api` prefix.
# All specific endpoint definitions (e.g., from `file_contents.py`) become accessible via `/api/...`.
app.include_router(api_router, prefix="/api")

# --- Root URL Redirect ---
# A convenience endpoint to redirect the base URL to the interactive API documentation.
@app.get("/")
async def redirect_to_docs():
    """
    Redirects the root URL (`/`) to the interactive API documentation (Swagger UI).
    """
    response = RedirectResponse(url="/docs")
    return response

# --- Startup Event Listener ---
# This function is executed once when the FastAPI application successfully starts up.
# It can be used for initializing resources, logging, etc.
@app.on_event("startup")
async def startup_event():
    # Print a confirmation message indicating the API server is listening.
    print(f"NexusMind AI API starting up on http://{settings.APP_HOST}:{settings.APP_PORT}")
```

```json Response Example
{"success":true}
```

# Empowering Your Digital World: File Ingestion & Intelligent

<!-- python@ -->

Analysis
Recipe Goal: Unlock the hidden intelligence within your digital files by securely ingesting them into NexusMind AI, where they become living, analyzable information assets and immediately embark on their journey into the immutable Intelli-Content Ledger.
File Names Impacted by This Recipe:
This single, comprehensive recipe integrates changes and definitions across the very core of your NexusMind AI backend. It directly involves:
src/main.py (The main application startup)
src/config.py (Core settings)
src/database.py (Database connection and ORM setup)
src/models.py (All database table definitions for FileContent, Analysis, and Semantic Ledger)
src/schemas.py (All API data input/output contracts)
src/services/file_ingestion.py (Handles file storage to disk/cloud)
src/services/ai_analysis.py (Simulates AI processing to extract insights)
src/tasks/celery_tasks.py (Defines background tasks for heavy lifting)
src/api/routers.py (API route aggregation)
src/api/v1/file_contents.py (The specific API endpoints for file management)
The NexusMind AI Story: From Data Silence to Intelligent Insight
Imagine a typical afternoon. Across your organization, new reports, legal briefs, scientific papers, images, and audio recordings are constantly being created, revised, and shared. For years, these digital files lived in digital silos—directories, cloud drives, email attachments. They held immense value, but their insights remained locked away, hidden unless someone manually opened and read them. The "context" was trapped, waiting to be rediscovered.
This is where the vision of NexusMind AI began to breathe. We imagined a world where files aren't just inert objects, but active participants in an evolving knowledge system. We wanted to move from managing "documents" to orchestrating "insights."
The heart of this transformation is encapsulated in our first, foundational recipe: "Empowering Your Digital World: File Ingestion & Intelligent Analysis."
The "Moment of Upload": The Genesis of Insight
It starts simply enough: you upload a file. Maybe it's that verbose 100-page research paper, or a dense financial spreadsheet, or a photograph from a recent inspection.
Traditionally, this would mean storage. For NexusMind AI, it's the moment of genesis for deep understanding.
A New Digital Being is Born (POST /api/file_contents):
As you hit 'upload', NexusMind AI springs into action. Our API doesn't just receive raw data; it immediately recognizes your file as a new FileContent resource, a fundamental entity within our system. This is where we create a unique digital fingerprint for it. At the very same instant, a dedicated "Content Account" is opened for your file within our revolutionary Intelli-Content Ledger. Think of this as your file getting its own immutable journal, ready to log every single insight it ever yields.
The Journey Begins: Ingestion & Queued for Brilliance (Asynchronous Processing):
Once acknowledged, your file doesn't bottleneck the system. NexusMind AI leverages advanced asynchronous processing. If you uploaded raw bytes, they're whisked away to secure storage. If you provided an external URL, a powerful background service begins fetching it. In either case, your API call is swiftly accepted with a 202 Accepted response, providing you with an operation_id to track its invisible journey. In the background, like a meticulous librarian preparing for research, an AnalysisJob is quietly created and immediately queued up for our powerful AI.
The AI Awakening: Dissection and Discovery:
Our specialized AI analysis engines wake up to your file. Depending on the Analysis Profile you've chosen (perhaps one tuned for legal sentiment or object detection in images), our AI begins its meticulous work. It doesn't just skim; it dissects. It extracts keywords, identifies named entities (people, organizations), quantifies sentiment, classifies topics, recognizes objects in images, or even transcribes audio.
The Immutable Record: Semantic Ledger Entries (The Core Magic!):
This is where the Intelli-Content Ledger truly shines. Every single discovery the AI makes – each keyword, each entity, each sentiment score – isn't merely an attribute. It becomes a SemanticLedgerEntry. These are immutable, timestamped "facts" explicitly logged to your file's "Content Account." They even link back to the specific SemanticLedgerTransaction (the unique AI analysis run that generated them), giving you unparalleled auditability of when and how a particular insight was recorded. And for concepts (like "Artificial Intelligence Ethics"), NexusMind AI simultaneously logs additional SemanticLedgerEntry objects to dedicated "Concept Accounts," automatically building a powerful, cross-document knowledge graph.
What This Means for You:
Auditability Redefined: Every AI-derived insight has an unshakeable provenance. You know what was extracted, when, from where in the document, and by which version of the AI.
A Living Knowledge Base: Your files are no longer inert. They are constantly enriching the Intelli-Content Ledger with verifiable, actionable insights, building an interconnected knowledge network that reveals relationships you never saw.
Responsive Experience: While complex AI churns in the background, your API interactions remain lightning-fast thanks to our asynchronous architecture.
Foundational Power: This recipe establishes the core capabilities that will allow you to list files, retrieve their latest status, and even permanently delete them and all their derived knowledge when no longer needed.
This isn't just data management; it's intelligence management, powered by a new paradigm in digital asset understanding.