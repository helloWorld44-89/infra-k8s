---
title: AudioFile Architecture Guide
date: 2026-05-29
author: Hermes Docs Agent
tags: [architecture, documentation, audiobook, docker]
---

# AudioFile Architecture Guide

## Overview

AudioFile is a self-hosted, containerized audiobook management platform that provides a complete solution for importing, converting, organizing, and streaming audiobooks. The system is designed for personal and small-scale deployment with a focus on ease of use and comprehensive audiobook management features.

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    AudioFile Container                      │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────┐ │
│  │   FastAPI        │  │   React Web UI  │  │   Nginx    │ │
│  │   Backend        │  │   Frontend     │  │   Proxy    │ │
│  └─────────────────┘  └─────────────────┘  └────────────┘ │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────┐ │
│  │   SQLite        │  │   Celery       │  │   Redis    │ │
│  │   Database      │  │   Queue        │  │   Cache    │ │
│  └─────────────────┘  └─────────────────┘  └────────────┘ │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │   DRM Service   │  │ Conversion     │                   │
│  │   (RCrack)      │  │ Engine        │                   │
│  └─────────────────┘  └─────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. Backend Services

**FastAPI Backend**
- RESTful API server written in Python 3.11+
- Handles all business logic and data operations
- Provides endpoints for books, users, conversions, and imports
- Integrated with CORS for frontend communication
- Health monitoring endpoint for system status

**Database Layer**
- **SQLite**: Primary database for metadata storage
- File-based database for easy backup and portability
- SQLAlchemy ORM for database operations
- Supports all book, user, and conversion job data

**Task Queue System**
- **Celery**: Background job processing for audio conversions
- **Redis**: Message broker and cache for queue management
- Asynchronous processing of long-running conversion tasks
- Progress tracking and error handling

#### 2. Frontend Application

**React + TypeScript Frontend**
- Modern React 18 application with TypeScript
- Bootstrap CSS for consistent UI styling
- Zustand for state management
- Responsive design for desktop and mobile

**Key UI Components**
- **Library Browser**: Grid/list view of audiobooks with filtering
- **Book Cards**: Individual book display with cover art and metadata
- **Audio Player**: Chapter-aware player with speed controls
- **Import Interface**: File upload and watch folder management
- **User Management**: Multi-user support with OIDC authentication

#### 3. Media Processing Engine

**Conversion Service**
- **FFmpeg**: Core audio processing library
- Support for multiple formats: M4B, MP3, FLAC, Opus
- Quality profiles for different use cases
- Chapter preservation and metadata embedding

**DRM Handling**
- **RCrack**: Audible AAX/AAXC decryption
- Activation byte management
- Secure DRM removal process

**Metadata Service**
- **Audible API**: Primary metadata source
- **Google Books**: Fallback for ASIN lookup
- **Open Library**: Additional metadata source
- Automatic metadata enrichment and validation

#### 4. File Management System

**Watch Folder Service**
- **Inotify**: Real-time file system monitoring
- **Cron**: Fallback scheduling for detection
- Automatic import of new audiobook files
- Support for multiple import directories

**File Organization**
- Configurable naming templates
- Series and sequence numbering support
- Cover art extraction and embedding
- Library structure maintenance

### Data Flow Architecture

#### Import Pipeline

```
1. File Detection → 2. Format Analysis → 3. DRM Check → 4. Metadata Lookup
     ↓                  ↓                  ↓                  ↓
5. DRM Removal → 6. Conversion → 7. Chapter Processing → 8. Library Update
```

#### Conversion Process

1. **Queue Job**: Conversion request added to Celery queue
2. **File Analysis**: Source format and quality detection
3. **DRM Processing**: Remove DRM if present
4. **Format Conversion**: Target format encoding with selected profile
5. **Metadata Embedding**: Cover art and chapter information
6. **File Organization**: Naming template and directory structure
7. **Database Update**: Update book record with new file location

### API Architecture

#### REST API Endpoints

**Book Management**
- `GET /api/books` - List books with pagination and filtering
- `GET /api/books/{id}` - Get detailed book information
- `POST /api/books` - Create new book record
- `PUT /api/books/{id}` - Update book metadata
- `DELETE /api/books/{id}` - Remove book from library

**Conversion Management**
- `GET /api/conversions` - List conversion jobs with status
- `POST /api/conversions` - Create new conversion job
- `GET /api/conversions/{id}` - Get conversion job details
- `PUT /api/conversions/{id}/cancel` - Cancel running job

**User Management**
- `GET /api/users/me` - Get current user profile
- `PUT /api/users/me` - Update user preferences
- `GET /api/users/{id}` - Admin: get user details

**Import Operations**
- `POST /api/books/import` - Trigger manual import
- `GET /api/import/status` - Check import progress
- `POST /webhooks/import-complete` - Import completion webhook

#### Webhook Support

- **Import Complete**: Triggered when import finishes
- **Conversion Complete**: Triggered when conversion finishes
- **Playback Action**: User interaction tracking

### Security Architecture

#### Authentication & Authorization
- **OIDC Integration**: Support for Authelia/Authentik
- **JWT Tokens**: Session management
- **Role-based Access**: Admin, Standard, Guest roles
- **CORS Protection**: Configurable origin policies

#### Data Security
- **Environment Variables**: All configuration external from code
- **Input Validation**: Sanitization on all API endpoints
- **File Permissions**: Proper user/group ownership
- **Encryption**: Support for encrypted audio files

### Deployment Architecture

#### Containerization
- **Single Docker Image**: Multi-arch support (AMD64 + ARM64)
- **Multi-stage Build**: Optimized image size
- **Non-root User**: Security best practices
- **Health Checks**: Container health monitoring

#### Orchestration
- **Docker Compose**: Simple deployment orchestration
- **Volume Mounts**: Library, config, inbox, logs
- **Environment Configuration**: Runtime configuration
- **Reverse Proxy**: Nginx included for HTTPS termination

#### Resource Management
- **Memory**: Configurable limits for conversion processes
- **Storage**: Volume mounting for large media libraries
- **CPU**: Priority handling for conversion jobs
- **Network**: Port mapping and service discovery

### Integration Points

#### External Services
- **Audible API**: Metadata and catalog information
- **Google Books**: Fallback metadata source
- **Open Library**: Additional enrichment data
- **OIDC Provider**: User authentication

#### File System Integration
- **Watch Folders**: Automatic file detection
- **Library Structure**: Organized directory hierarchy
- **Backup Support**: Database and media file backup
- **Network Storage**: SMB/NFS support for remote libraries

### Monitoring & Observability

#### Health Monitoring
- **Health Endpoint**: `/health` for container monitoring
- **Database Connection**: Connection health checks
- **Queue Status**: Celery worker monitoring
- **Service Dependencies**: External service availability

#### Logging
- **Application Logs**: Structured logging for debugging
- **Conversion Logs**: Detailed progress tracking
- **Error Logging**: Error capture and notification
- **Access Logs**: API access and authentication logging

### Performance Considerations

#### Concurrency
- **Celery Workers**: Configurable worker count
- **Database Connection Pool**: Optimized connection management
- **File Processing**: Parallel processing support
- **Memory Management**: Efficient resource usage

#### Scalability
- **Horizontal Scaling**: Multiple container instances
- **Load Balancing**: Nginx reverse proxy configuration
- **Database Scaling**: SQLite with proper indexing
- **Cache Management**: Redis for frequent data access

### Backup & Recovery

#### Database Backup
- **File-based**: SQLite database file backup
- **Automated**: Scheduled backup scripts
- **Version Control**: Git integration for metadata changes
- **Export/Import**: Database migration tools

#### Media Backup
- **Library Structure**: Complete library directory backup
- **Incremental**: Differential backup support
- **Verification**: Backup integrity checking
- **Restoration**: Automated restore procedures

### Configuration Management

#### Environment Variables
- **Database Configuration**: Connection strings and credentials
- **Redis Configuration**: Queue and cache settings
- **OIDC Configuration**: Authentication provider settings
- **Application Settings**: Feature flags and behavior control

#### File Configuration
- **Naming Templates**: Custom file naming patterns
- **Quality Profiles**: Conversion quality presets
- **Watch Folders**: Import directory configuration
- **User Settings**: Default user preferences

## Technology Stack

### Backend Technologies
- **Python 3.11+**: Primary programming language
- **FastAPI**: Web framework for REST API
- **SQLAlchemy**: ORM for database operations
- **Celery**: Background task processing
- **Redis**: Message broker and cache
- **FFmpeg**: Audio processing library
- **Mutagen**: Audio metadata handling
- **Pydub**: Audio manipulation library

### Frontend Technologies
- **React 18**: User interface framework
- **TypeScript**: Type-safe JavaScript
- **Bootstrap CSS**: UI component library
- **Zustand**: State management
- **Vite**: Build tool and development server

### Infrastructure Technologies
- **Docker**: Containerization platform
- **Docker Compose**: Orchestration tool
- **Nginx**: Reverse proxy and web server
- **SQLite**: File-based database
- **Redis**: In-memory data store

## Development Guidelines

### Code Structure
- **Modular Architecture**: Clear separation of concerns
- **Service Layer**: Business logic abstraction
- **Repository Pattern**: Data access layer
- **Middleware**: Cross-cutting concerns handling

### Testing Strategy
- **Unit Tests**: Individual component testing
- **Integration Tests**: API endpoint testing
- **Bruno Tests**: API contract testing
- **E2E Tests**: Full workflow testing

### Documentation Standards
- **API Documentation**: OpenAPI/Swagger integration
- **Code Comments**: Comprehensive inline documentation
- **README Files**: Project and component documentation
- **Architecture Guides**: High-level design documentation

This architecture provides a solid foundation for a scalable, maintainable audiobook management system with comprehensive features for personal and small-scale deployment.