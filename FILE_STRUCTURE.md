# Interview Chamber - File Structure

## Complete Directory Tree

```
interview-chamber/
│
├── interview_chamber/              # Main package
│   ├── __init__.py
│   ├── main.py                    # CLI entry point
│   │
│   ├── config/                    # Configuration management
│   │   ├── __init__.py
│   │   ├── loader.py              # YAML config loader with validation
│   │   └── schema.py              # Pydantic models for config validation
│   │
│   ├── llm/                       # LLM client abstraction
│   │   ├── __init__.py
│   │   ├── client.py              # Base abstract LLM client interface
│   │   ├── providers.py           # Provider implementations (OpenAI, Anthropic, etc.)
│   │   └── manager.py             # Dual connection manager
│   │
│   ├── context/                   # Context isolation system
│   │   ├── __init__.py
│   │   ├── isolation.py           # Context masking and isolation logic
│   │   └── messages.py            # Message structure and management
│   │
│   ├── extraction/                # Thought extraction engine
│   │   ├── __init__.py
│   │   └── thoughts.py            # Dynamic thought extraction with pattern matching
│   │
│   ├── logging/                   # Logging system
│   │   ├── __init__.py
│   │   └── manager.py             # Timestamped file logging manager
│   │
│   └── utils/                     # Utility functions
│       ├── __init__.py
│       └── patterns.py            # Pattern matching utilities
│
├── configs/                       # Configuration files (user-provided)
│   ├── prime_config.yaml          # Prime LLM configuration
│   └── candidate_config.yaml      # Candidate LLM configuration
│
├── logs/                          # Generated at runtime (gitignored)
│   ├── prime_log_YYYYMMDD_HHMMSS.txt
│   ├── candidate_log_YYYYMMDD_HHMMSS.txt
│   └── shared_transcript_YYYYMMDD_HHMMSS.txt
│
├── tests/                         # Test suite (future)
│   ├── __init__.py
│   ├── test_extraction.py
│   ├── test_isolation.py
│   └── test_integration.py
│
├── requirements.txt               # Python dependencies
├── README.md                      # Project documentation
├── FILE_STRUCTURE.md              # This file
├── implementation_plan.md         # Detailed implementation plan
└── .gitignore                     # Git ignore rules
```

## Module Responsibilities

### `interview_chamber/main.py`
- CLI interface using Click
- Argument parsing (config paths, max turns, output dir)
- Main interview loop orchestration
- Error handling and graceful shutdown

### `interview_chamber/config/`
- **`loader.py`**: Loads YAML configs, validates against schema, handles env var substitution
- **`schema.py`**: Pydantic models defining config structure (LLMConfig, SystemPromptConfig, etc.)

### `interview_chamber/llm/`
- **`client.py`**: Abstract base class for LLM clients
- **`providers.py`**: Concrete implementations (OpenAIClient, AnthropicClient)
- **`manager.py`**: Manages two independent client instances, handles connection pooling

### `interview_chamber/context/`
- **`messages.py`**: Message data structure with speaker, content, metadata
- **`isolation.py`**: Core logic for building Prime vs Candidate contexts with thought masking

### `interview_chamber/extraction/`
- **`thoughts.py`**: Pattern-based extraction engine, handles multiple pattern types, nested tags

### `interview_chamber/logging/`
- **`manager.py`**: File-based logging with timestamps, three separate log files

### `interview_chamber/utils/`
- **`patterns.py`**: Regex compilation, pattern matching utilities

## Data Flow

```
CLI (main.py)
    ↓
Config Loader → Prime Config + Candidate Config
    ↓
LLM Manager → Prime Client + Candidate Client
    ↓
Context Isolator → Shared Conversation Buffer
    ↓
Thought Extractor → Extract & Mask Thoughts
    ↓
Log Manager → Three Timestamped Log Files
```

## Key Design Patterns

1. **Strategy Pattern**: Different LLM providers implement the same client interface
2. **Factory Pattern**: Config loader creates appropriate client instances
3. **Observer Pattern**: Log manager observes conversation events
4. **Builder Pattern**: Context isolator builds different context views

