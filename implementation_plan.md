# Interview Chamber - Implementation Plan

## Architecture Overview

Interview Chamber is a dual-LLM evaluation system where a **Prime** LLM interviews a **Candidate** LLM. The system maintains strict context isolation while enabling dynamic thought extraction and comprehensive logging.

---

## 1. File Structure

```
interview_chamber/
├── __init__.py
├── main.py                 # CLI entry point
├── config/
│   ├── __init__.py
│   ├── loader.py          # Configuration file loader
│   └── schema.py          # Pydantic models for config validation
├── llm/
│   ├── __init__.py
│   ├── client.py          # Base LLM client interface
│   ├── providers.py       # Provider-specific implementations (OpenAI, Anthropic, etc.)
│   └── manager.py         # Dual connection manager
├── context/
│   ├── __init__.py
│   ├── isolation.py       # Context masking and isolation logic
│   └── messages.py        # Message structure and management
├── extraction/
│   ├── __init__.py
│   └── thoughts.py        # Dynamic thought extraction engine
├── logging/
│   ├── __init__.py
│   └── manager.py         # Timestamped file logging
└── utils/
    ├── __init__.py
    └── patterns.py        # Pattern matching utilities

configs/
├── prime_config.yaml      # Prime LLM configuration
└── candidate_config.yaml  # Candidate LLM configuration

logs/                      # Generated at runtime
├── prime_log_<timestamp>.txt
├── candidate_log_<timestamp>.txt
└── shared_transcript_<timestamp>.txt

requirements.txt
README.md
.gitignore
```

---

## 2. Core Components

### 2.1 Configuration System (`config/`)

**Purpose:** Load and validate separate configuration files for Prime and Candidate.

**Key Files:**
- `config/schema.py`: Pydantic models defining the structure:
  ```python
  - LLMConfig: API endpoint, model name, temperature, max_tokens
  - SystemPromptConfig: System prompt content
  - ProviderConfig: API keys, base URLs, provider type
  - InterviewConfig: Overall interview settings
  ```

- `config/loader.py`: YAML loader that:
  - Validates configuration against schema
  - Handles environment variable substitution
  - Returns validated config objects

**Configuration File Format:**
```yaml
provider: "openai"  # or "anthropic", "custom"
api_key: "${OPENAI_API_KEY}"  # Supports env vars
base_url: "https://api.openai.com/v1"
model: "gpt-4-turbo-preview"
temperature: 0.7
max_tokens: 2000
system_prompt: |
  You are an interviewer evaluating a candidate...
thought_patterns:  # Optional: provider-specific patterns
  - "<thinking>"
  - "[reasoning]"
  - "<think>"
```

---

### 2.2 LLM Client System (`llm/`)

**Purpose:** Abstract API interactions and maintain two separate connections.

**Key Files:**
- `llm/client.py`: Base abstract class defining the interface:
  ```python
  class LLMClient(ABC):
      async def chat_completion(messages, **kwargs)
      async def stream_chat_completion(messages, **kwargs)
  ```

- `llm/providers.py`: Provider implementations:
  - `OpenAIClient`
  - `AnthropicClient`
  - `CustomClient` (for extensibility)

- `llm/manager.py`: **Dual Connection Manager**
  - Maintains two independent client instances
  - Handles connection pooling and retries
  - Manages API rate limits per connection

---

### 2.3 Context Isolation System (`context/`)

**Purpose:** Maintain separate conversation contexts for Prime and Candidate.

**Key Files:**
- `context/messages.py`: Message structure:
  ```python
  class Message:
      role: str  # "user", "assistant", "system"
      content: str
      timestamp: datetime
      metadata: dict  # For storing extracted thoughts
  ```

- `context/isolation.py`: **Core Context Masking Logic**

#### Context Isolation Algorithm:

1. **Shared Conversation Buffer:**
   - Maintains a single list of all messages exchanged
   - Each message tagged with `speaker: "prime" | "candidate"`

2. **Prime Context Builder:**
   ```
   Input: Shared conversation + Prime's own thoughts + (optional) Candidate's thoughts
   
   Algorithm:
   - Start with system prompt for Prime
   - For each message in shared conversation:
     - If speaker == "prime":
       - Add message with full content (including extracted thoughts if available)
     - If speaker == "candidate":
       - Add message with:
         * Public content (thoughts removed)
         * Optionally append Candidate's thoughts if `include_candidate_thoughts: true`
   - Return formatted messages for API call
   ```

3. **Candidate Context Builder:**
   ```
   Input: Shared conversation only
   
   Algorithm:
   - Start with system prompt for Candidate
   - For each message in shared conversation:
     - Add message with public content only (thoughts already stripped)
   - Return formatted messages for API call
   ```

4. **Thought Masking:**
   - Before adding Candidate messages to Prime context:
     - Extract thoughts using `extraction/thoughts.py`
     - Store thoughts in message metadata
     - Remove thought blocks from public content
   - Prime messages: Keep thoughts if `include_prime_thoughts: true`

**Implementation Details:**
```python
class ContextIsolator:
    def __init__(self, prime_config, candidate_config):
        self.prime_config = prime_config
        self.candidate_config = candidate_config
        self.shared_conversation = []
        self.thought_extractor = ThoughtExtractor(prime_config.thought_patterns)
    
    def build_prime_context(self, include_candidate_thoughts=False):
        messages = [{"role": "system", "content": self.prime_config.system_prompt}]
        for msg in self.shared_conversation:
            if msg.speaker == "prime":
                # Include full content with thoughts
                messages.append({
                    "role": "assistant",
                    "content": msg.full_content
                })
            elif msg.speaker == "candidate":
                # Public content only, optionally append thoughts
                public_content = msg.public_content
                if include_candidate_thoughts and msg.metadata.get("thoughts"):
                    public_content += f"\n[Candidate's internal thoughts: {msg.metadata['thoughts']}]"
                messages.append({
                    "role": "user",
                    "content": public_content
                })
        return messages
    
    def build_candidate_context(self):
        messages = [{"role": "system", "content": self.candidate_config.system_prompt}]
        for msg in self.shared_conversation:
            if msg.speaker == "prime":
                messages.append({
                    "role": "user",
                    "content": msg.public_content  # Thoughts already stripped
                })
            elif msg.speaker == "candidate":
                messages.append({
                    "role": "assistant",
                    "content": msg.public_content
                })
        return messages
    
    def add_message(self, speaker, raw_content):
        # Extract thoughts before storing
        public_content, thoughts = self.thought_extractor.extract(raw_content)
        
        message = Message(
            speaker=speaker,
            full_content=raw_content,
            public_content=public_content,
            metadata={"thoughts": thoughts}
        )
        self.shared_conversation.append(message)
        return message
```

---

### 2.4 Dynamic Thought Extraction (`extraction/`)

**Purpose:** Identify and extract "internal thoughts" from model outputs using configurable patterns.

**Key File:** `extraction/thoughts.py`

#### Thought Extraction Algorithm:

1. **Pattern Configuration:**
   - Load patterns from config files (provider-specific or global)
   - Support multiple pattern types:
     - XML-style: `<thinking>...</thinking>`, `<think>...</think>`
     - Bracket-style: `[reasoning]...[/reasoning]`, `[thoughts]...[/thoughts]`
     - Markdown-style: `:::thinking\n...\n:::`
     - Custom regex patterns

2. **Extraction Process:**
   ```python
   class ThoughtExtractor:
       def __init__(self, patterns):
           # Compile patterns into regex or parser
           self.patterns = self._compile_patterns(patterns)
       
       def extract(self, content: str) -> tuple[str, str]:
           """
           Returns: (public_content, thoughts)
           """
           thoughts = []
           public_content = content
           
           for pattern in self.patterns:
               matches = pattern.findall(content)
               for match in matches:
                   # Extract thought block
                   thought_text = self._extract_thought_text(match)
                   thoughts.append(thought_text)
                   # Remove from public content
                   public_content = pattern.sub('', public_content)
           
           # Clean up extra whitespace
           public_content = self._clean_whitespace(public_content)
           combined_thoughts = '\n\n'.join(thoughts)
           
           return public_content, combined_thoughts
   ```

3. **Pattern Matching Strategy:**
   - **Nested Tag Support:** Handle nested tags correctly
   - **Multi-line Support:** Patterns can span multiple lines
   - **Priority Order:** Process patterns in order (first match wins for overlapping patterns)
   - **Fallback:** If no patterns match, return original content with empty thoughts

4. **Pattern Compilation:**
   ```python
   def _compile_patterns(self, pattern_configs):
       compiled = []
       for pattern_config in pattern_configs:
           if isinstance(pattern_config, str):
               # Simple tag pattern: "<thinking>...</thinking>"
               tag = pattern_config.strip('<>[]')
               regex = re.compile(
                   rf'<{tag}>(.*?)</{tag}>',
                   re.DOTALL | re.IGNORECASE
               )
           elif isinstance(pattern_config, dict):
               # Advanced pattern with custom regex
               regex = re.compile(
                   pattern_config['regex'],
                   pattern_config.get('flags', 0)
               )
           compiled.append(regex)
       return compiled
   ```

5. **Edge Cases:**
   - **Unclosed Tags:** Log warning, extract partial content
   - **Overlapping Tags:** Use priority order, extract all matches
   - **Nested Tags:** Extract inner-most tag first, then outer
   - **Empty Thoughts:** Skip empty thought blocks

---

### 2.5 Logging System (`logging/`)

**Purpose:** Write timestamped logs to three separate files.

**Key File:** `logging/manager.py`

**Implementation:**
```python
class LogManager:
    def __init__(self, base_dir="logs"):
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        self.prime_log = Path(base_dir) / f"prime_log_{timestamp}.txt"
        self.candidate_log = Path(base_dir) / f"candidate_log_{timestamp}.txt"
        self.shared_log = Path(base_dir) / f"shared_transcript_{timestamp}.txt"
        
        # Initialize files with headers
        self._initialize_logs()
    
    def log_prime(self, message, include_thoughts=True):
        """Log Prime's message with optional thoughts"""
        with open(self.prime_log, 'a') as f:
            f.write(f"[{datetime.now().isoformat()}] PRIME:\n")
            f.write(f"{message.full_content if include_thoughts else message.public_content}\n\n")
    
    def log_candidate(self, message, include_thoughts=True):
        """Log Candidate's message with optional thoughts"""
        with open(self.candidate_log, 'a') as f:
            f.write(f"[{datetime.now().isoformat()}] CANDIDATE:\n")
            f.write(f"{message.full_content if include_thoughts else message.public_content}\n\n")
    
    def log_shared(self, message):
        """Log to shared transcript (public content only)"""
        with open(self.shared_log, 'a') as f:
            f.write(f"[{datetime.now().isoformat()}] {message.speaker.upper()}:\n")
            f.write(f"{message.public_content}\n\n")
```

---

### 2.6 Main Orchestration (`main.py`)

**Purpose:** CLI interface and interview loop.

**Flow:**
1. Load configurations (Prime and Candidate)
2. Initialize LLM clients (two separate connections)
3. Initialize context isolator and thought extractor
4. Initialize log manager
5. **Interview Loop:**
   ```
   while not interview_complete:
       # Prime's turn
       prime_context = isolator.build_prime_context()
       prime_response = await prime_client.chat_completion(prime_context)
       prime_message = isolator.add_message("prime", prime_response)
       log_manager.log_prime(prime_message)
       log_manager.log_shared(prime_message)
       
       # Candidate's turn
       candidate_context = isolator.build_candidate_context()
       candidate_response = await candidate_client.chat_completion(candidate_context)
       candidate_message = isolator.add_message("candidate", candidate_response)
       log_manager.log_candidate(candidate_message)
       log_manager.log_shared(candidate_message)
       
       # Check termination conditions
       if should_terminate(prime_response, candidate_response):
           break
   ```

---

## 3. Implementation Details

### 3.1 Error Handling

- **API Failures:** Retry with exponential backoff per connection
- **Config Errors:** Validate at startup, provide clear error messages
- **Pattern Matching Failures:** Log warning, continue with original content
- **Logging Failures:** Fail gracefully, continue interview

### 3.2 Performance Considerations

- **Async/Await:** Use async for all API calls to enable concurrent requests
- **Streaming Support:** Optional streaming for real-time output
- **Connection Pooling:** Reuse HTTP connections per provider

### 3.3 Extensibility

- **New Providers:** Add new client class inheriting from `LLMClient`
- **Custom Patterns:** Support regex patterns in config
- **Plugins:** Architecture allows for custom thought extractors

---

## 4. Testing Strategy

1. **Unit Tests:**
   - Thought extraction with various patterns
   - Context isolation logic
   - Configuration loading and validation

2. **Integration Tests:**
   - End-to-end interview flow
   - Dual connection management
   - Logging verification

3. **Mock Tests:**
   - Mock API responses to test isolation
   - Simulate various thought patterns

---

## 5. Configuration Examples

### `configs/prime_config.yaml`
```yaml
provider: "openai"
api_key: "${OPENAI_API_KEY}"
model: "gpt-4-turbo-preview"
temperature: 0.8
max_tokens: 2000
system_prompt: |
  You are an expert interviewer evaluating a candidate LLM.
  Ask probing questions and assess the candidate's responses.
  
  You may see the candidate's internal thoughts if they are available.
  Use this information to ask more targeted questions.
  
  Format your internal reasoning using <thinking>...</thinking> tags.
thought_patterns:
  - "<thinking>"
  - "[reasoning]"
include_candidate_thoughts: true
include_prime_thoughts: true
```

### `configs/candidate_config.yaml`
```yaml
provider: "anthropic"
api_key: "${ANTHROPIC_API_KEY}"
model: "claude-3-opus-20240229"
temperature: 0.7
max_tokens: 2000
system_prompt: |
  You are a candidate being interviewed for a position.
  Answer questions thoughtfully and demonstrate your capabilities.
  
  Use <think>...</think> tags for internal thoughts.
thought_patterns:
  - "<think>"
  - "[thoughts]"
```

---

## 6. CLI Interface

```bash
interview-chamber \
  --prime-config configs/prime_config.yaml \
  --candidate-config configs/candidate_config.yaml \
  --max-turns 20 \
  --output-dir logs \
  --verbose
```

---

## 7. Future Enhancements

- **Streaming Mode:** Real-time output to terminal
- **Evaluation Metrics:** Automatic scoring of responses
- **Multi-round Interviews:** Support for multiple interview sessions
- **Web UI:** Optional web interface for monitoring
- **Export Formats:** JSON, Markdown export options

