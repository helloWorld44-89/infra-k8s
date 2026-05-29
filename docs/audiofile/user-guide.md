---
title: AudioFile User Guide
date: 2026-05-29
author: Hermes Docs Agent
tags: [user-guide, documentation, audiobook, deployment]
---

# AudioFile User Guide

## Introduction

AudioFile is a self-hosted audiobook management platform that allows you to import, convert, organize, and stream your audiobook collection. This guide will help you get started with using AudioFile effectively.

## Getting Started

### First Time Setup

1. **Install AudioFile** using the deployment guide
2. **Access the Web Interface** at `http://your-server:8080`
3. **Configure Your Library** by setting up watch folders
4. **Import Your Audiobooks** using auto-import or manual upload

### Dashboard Overview

When you first access AudioFile, you'll see:

- **Library Browser**: Your audiobook collection
- **Quick Actions**: Import, convert, and organize buttons
- **Recent Activity**: Latest imports and conversions
- **Storage Status**: Disk usage and library size

## Managing Your Audiobook Library

### Importing Audiobooks

#### Method 1: Auto-Import (Recommended)

Auto-import automatically detects new audiobook files in your watch folders:

1. **Configure Watch Folders**: Go to Settings → Import
2. **Add Watch Directories**: Add folders where you store new audiobooks
3. **Set Naming Patterns**: Configure how files should be organized
4. **Enable Auto-Import**: Toggle auto-import on

#### Method 2: Manual Import

Manually import specific files:

1. Click **"Import Audiobooks"** on the dashboard
2. Drag and drop files or click to browse
3. Select files and click **"Import"**
4. Wait for processing to complete

#### Supported Formats

- **Input**: AAX, AAXC (DRM-protected), M4B, MP3, FLAC, OPUS, WAV
- **Output**: M4B, MP3, FLAC, OPUS
- **DRM Support**: AAX/AAXC with activation bytes or auth codes

### Organizing Your Library

#### Book Views

AudioFile provides multiple ways to view your library:

- **Grid View**: Cover-focused visual browsing
- **List View**: Detailed text-based browsing
- **Series View**: Grouped by series
- **Author View**: Grouped by author

#### Filtering and Searching

Use the search and filter options to find specific books:

- **Search**: Title, author, narrator, series
- **Filter**: By format, duration, date added, status
- **Sort**: By title, author, date added, duration

#### Book Details

Click on any book to see detailed information:

- **Metadata**: Title, author, narrator, series, genre
- **File Information**: Format, size, duration, quality
- **Playback**: Chapters, bookmarks, progress
- **Actions**: Convert, download, delete

### Listening to Audiobooks

#### Web Player

AudioFile includes a built-in web player for streaming:

- **Chapter Navigation**: Skip between chapters
- **Playback Controls**: Play/pause, skip forward/backward
- **Speed Control**: Adjust playback speed (0.5x - 3.0x)
- **Volume Control**: Adjustable volume levels
- **Progress Bar**: Visual progress tracking

#### Mobile Support

AudioFile works on mobile devices:

- **Responsive Design**: Optimized for phones and tablets
- **Streaming**: Listen directly from your device
- **Bookmarks**: Sync across devices (if auth enabled)

## Converting Audiobooks

### Conversion Process

Convert audiobooks to different formats for compatibility:

1. **Select Book**: Choose a book from your library
2. **Choose Format**: Select output format (M4B, MP3, FLAC, OPUS)
3. **Set Quality**: Choose quality profile
4. **Start Conversion**: Click "Convert" button

### Quality Profiles

| Format | Quality Profile | Description | Use Case |
|--------|----------------|-------------|----------|
| M4B | High | 192kbps, chapters | iPhone/iPod |
| M4B | Medium | 128kbps, chapters | General use |
| MP3 | High | 320kbps | Best quality |
| MP3 | Medium | 192kbps | Standard |
| FLAC | Lossless | Uncompressed | Audio quality |
| OPUS | High | 128kbps | Small files |

### DRM Removal

AudioFile can remove DRM from Audible books:

1. **Provide Activation Bytes**: Enter 16-character activation code
2. **Provide Auth Code**: Enter Audible authentication code
3. **Process**: System will decrypt and convert
4. **Verify**: Check decrypted file quality

## User Management

### Creating Users

Multi-user support allows family members to have separate libraries:

1. **Admin Access**: Go to Settings → Users
2. **Add User**: Click "Add User" button
3. **Set Permissions**: Choose user role (Admin, Standard, Guest)
4. **Configure Profile**: Set name, preferences

### User Roles

| Role | Permissions | Use Case |
|------|-------------|----------|
| Admin | Full access, user management | Primary user |
| Standard | Full library access, conversions | Family members |
| Guest | Read-only access | Temporary users |

### User Preferences

Each user can customize their experience:

- **Theme**: Light/dark mode
- **Language**: Interface language
- **Playback**: Default speed, volume
- **Library**: Default view, filters

## Settings and Configuration

### General Settings

- **Library Path**: Where audiobooks are stored
- **Inbox Path**: Where imports are processed
- **Log Level**: Debug, Info, Warning, Error
- **Auto-cleanup**: Remove processed imports

### Import Settings

- **Watch Folders**: Directories to monitor
- **File Patterns**: Supported file extensions
- **Naming Templates**: How to organize files
- **Auto-process**: Automatically convert imported files

### Conversion Settings

- **Default Format**: Preferred output format
- **Quality Profiles**: Custom quality settings
- **Parallel Jobs**: Number of simultaneous conversions
- **Temporary Storage**: Where to store converted files

### Security Settings

- **Authentication**: OIDC/SSO integration
- **Access Control**: IP restrictions
- **Session Timeout**: Inactivity timeout
- **HTTPS**: SSL/TLS configuration

## Advanced Features

### OPDS Catalog

AudioFile provides an OPDS (Open Publication Distribution System) feed:

- **Access**: `http://your-server:8080/opds`
- **Compatible**: Most audiobook apps (OverDrive, Libby, etc.)
- **Content**: Complete library with metadata

### Webhooks

Automate workflows with webhooks:

- **Import Complete**: Trigger when import finishes
- **Conversion Complete**: Trigger when conversion finishes
- **Playback Action**: Track user interactions
- **Custom Endpoints**: Send data to your services

### API Integration

Access AudioFile programmatically:

- **REST API**: Full API documentation available
- **Authentication**: API key or OIDC tokens
- **Rate Limiting**: Configurable request limits
- **Webhooks**: Event-driven integration

## Troubleshooting

### Common Issues

#### Import Problems

**Issue**: Files not importing automatically
**Solution**: Check watch folder permissions and file formats

**Issue**: DRM removal failing
**Solution**: Verify activation bytes are correct

**Issue**: Conversion taking too long
**Solution**: Reduce quality settings or limit parallel jobs

#### Playback Issues

**Issue**: Audio skipping or buffering
**Solution**: Check network connection and file quality

**Issue**: Player not working on mobile
**Solution**: Ensure HTTPS is enabled for mobile browsers

#### Library Problems

**Issue**: Books missing from library
**Solution**: Check database integrity and file locations

**Issue**: Search not finding books
**Solution**: Rebuild search index or check metadata

### Getting Help

- **Documentation**: Check this guide and API docs
- **Logs**: Check `/logs` directory for error details
- **Community**: Join the project community for support
- **Issues**: Report bugs on the project GitHub

## Best Practices

### Library Organization

- Use consistent naming conventions
- Organize by series and author
- Regular backups of metadata
- Monitor storage space usage

### Performance Optimization

- Use appropriate quality settings
- Limit parallel conversions
- Regular maintenance of database
- Monitor system resources

### Security

- Keep software updated
- Use strong authentication
- Regular backups of library
- Monitor access logs

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Space` | Play/Pause |
| `→` | Skip forward 30s |
| `←` | Skip backward 30s |
| `↑` | Volume up |
| `↓` | Volume down |
| `Home` | Go to library |
| `Ctrl+F` | Search |
| `Esc` | Close modal |

## Mobile Tips

- Use browser bookmark for quick access
- Enable desktop mode for better experience
- Download audiobooks for offline listening
- Use headphones for better quality

---

*This documentation is part of the AudioFile project. For the latest updates and additional resources, visit the project repository.*