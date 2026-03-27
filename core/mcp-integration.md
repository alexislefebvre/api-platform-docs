# Model Context Protocol (MCP) Integration

API Platform provides first-class integration with the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/), allowing you to expose your APIs as **tools** and **resources** for Large Language Models (LLMs) and AI agents.

MCP is an open protocol standardizing how AI assistants connect to data sources and tools. By integrating MCP with API Platform, you can make your API operations discoverable and executable by AI systems like Claude, ChatGPT, or any MCP-compatible client.

## Table of Contents

* [Introduction](#introduction)
* [Installation](#installation)
* [Basic Usage](#basic-usage)
  * [Exposing Tools](#exposing-tools)
  * [Exposing Resources](#exposing-resources)
* [Advanced Features](#advanced-features)
  * [Custom Processors](#custom-processors)
  * [Structured Content](#structured-content)
  * [Tool Annotations](#tool-annotations)
* [Configuration](#configuration)
* [Best practices](#best-practices)

## Introduction

The MCP integration in API Platform enables two main capabilities:

1. **Tools** - API operations that AI agents can invoke to perform actions (e.g., create a book, search products, send notifications)
2. **Resources** - Data sources that AI agents can read (e.g., documentation, configuration files, datasets)

When you expose an API operation through MCP:
* It becomes automatically discoverable by MCP clients
* Input/output schemas are generated from your PHP classes
* Validation and serialization happen automatically
* You can leverage all API Platform features (providers, processors, security, etc.)

## Installation

First, install the MCP component:

```bash
composer require api-platform/mcp
```

The MCP integration requires:
* PHP 8.2 or higher
* API Platform 4.2 or higher
* The `mcp/sdk` package (installed automatically)

For Symfony applications, the bundle is automatically registered if you have Symfony Flex enabled.

## Basic Usage

### Exposing Tools

A **tool** is an executable operation that AI agents can call with specific parameters. To expose a tool, use the `#[McpTool]` attribute:

```php
<?php
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\McpTool;

#[ApiResource(
    operations: [],
    mcp: [
        'create_book' => new McpTool(
            description: 'Creates a new book in the library'
        ),
    ]
)]
class BookCreator
{
    public function __construct(
        public string $title,
        public string $author,
        public ?string $isbn = null,
        public ?int $publicationYear = null,
    ) {}
}
```

This exposes a `create_book` tool that AI agents can invoke. The tool's input schema is automatically generated from your class properties and their types.

**How it works:**
1. The AI agent discovers the `create_book` tool with its JSON Schema
2. The agent calls the tool with arguments like `{"title": "1984", "author": "George Orwell"}`
3. API Platform maps the arguments to your `BookCreator` class
4. Your processor (or default state machine) handles the operation
5. The result is serialized and returned to the agent

### Exposing Resources

A **resource** is read-only content that AI agents can access. Use the `#[McpResource]` attribute:

```php
<?php
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\McpResource;

#[ApiResource(
    operations: [],
    mcp: [
        'api_docs' => new McpResource(
            uri: 'resource://my-app/api-documentation',
            name: 'API Documentation',
            description: 'Complete API reference and guides',
            mimeType: 'text/markdown',
            provider: [self::class, 'provide']
        ),
    ]
)]
class ApiDocumentation
{
    public function __construct(
        public string $content,
        public string $lastUpdated,
    ) {}

    public static function provide(): self
    {
        return new self(
            content: file_get_contents(__DIR__ . '/../../docs/api.md'),
            lastUpdated: date('Y-m-d H:i:s'),
        );
    }
}
```

**Key differences from tools:**
* Resources have a unique `uri` (required)
* Resources are read-only (no mutations)
* Resources typically provide static or slowly-changing content
* The `provider` parameter defines how to fetch the resource content

## Advanced Features

### Custom Processors

You can use custom processors to handle tool invocations with specific business logic:

```php
<?php
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\McpTool;
use Mcp\Schema\Content\TextContent;
use Mcp\Schema\Result\CallToolResult;

#[ApiResource(
    operations: [],
    mcp: [
        'analyze_text' => new McpTool(
            description: 'Analyzes text and returns statistics',
            processor: [self::class, 'process']
        ),
    ]
)]
class TextAnalyzer
{
    public function __construct(
        public string $text,
        public bool $includeWordCount = true,
        public bool $includeSentimentAnalysis = false,
    ) {}

    public static function process(self $data): CallToolResult
    {
        $wordCount = str_word_count($data->text);
        $charCount = strlen($data->text);
        
        $result = "Word count: {$wordCount}\nCharacter count: {$charCount}";
        
        if ($data->includeSentimentAnalysis) {
            // Implement sentiment analysis...
            $result .= "\nSentiment: Positive";
        }
        
        return new CallToolResult(
            content: [new TextContent($result)],
            isError: false,
            meta: ['wordCount' => $wordCount, 'charCount' => $charCount]
        );
    }
}
```

The processor receives the mapped input object and can:
* Return a `CallToolResult` directly for full control
* Return any object to use default serialization
* Access all injected dependencies via service configuration

### Structured Content

By default, tools include structured content in their responses, allowing AI agents to parse the output programmatically. You can control this behavior:

```php
#[ApiResource(
    operations: [],
    mcp: [
        'search_products' => new McpTool(
            structuredContent: true, // default
            processor: [ProductSearchProcessor::class, 'search']
        ),
        'simple_echo' => new McpTool(
            structuredContent: false, // only text content
        ),
    ]
)]
class ProductSearch { /* ... */ }
```

When `structuredContent: true`:
* The response includes both text and structured JSON data
* AI agents can parse the structured data for further processing
* Uses API Platform's serialization system

When `structuredContent: false`:
* Only text content is returned
* Smaller response size
* Better for simple operations

### Tool Annotations

MCP supports annotations for rich metadata about tools. These help AI agents understand how to use your tools more effectively:

```php
#[ApiResource(
    operations: [],
    mcp: [
        'send_email' => new McpTool(
            description: 'Sends an email to a recipient',
            annotations: [
                'audience' => ['developers', 'administrators'],
                'priority' => 0.9,
            ],
            icons: [
                'https://example.com/icons/email.png',
            ],
            meta: [
                'rateLimit' => '100 per hour',
                'requiresAuth' => true,
            ]
        ),
    ]
)]
class EmailSender { /* ... */ }
```

**Annotation fields:**
* `annotations` - Hints for AI behavior (audience, priority, etc.)
* `icons` - Visual representations of the tool
* `meta` - Additional metadata (rate limits, auth requirements, etc.)

## Configuration

### Symfony Bundle Configuration

For Symfony applications, the MCP bundle is automatically configured. You can customize it in `config/packages/api_platform.yaml`:

```yaml
api_platform:
    mcp:
        enabled: true
        # Additional MCP-specific configuration
```

### Registering Tools and Resources

Tools and resources are automatically discovered from your API resources. The component scans all classes with `#[ApiResource]` attributes and registers any `mcp` operations.

## Best Practices

1. **Use Descriptive Names**: Tool names should clearly indicate their purpose
   ```php
   'search_products' => new McpTool(...) // Good
   'tool1' => new McpTool(...) // Bad
   ```

2. **Provide Clear Descriptions**: Help AI agents understand when to use your tools
   ```php
   description: 'Searches for products by name, category, or SKU. Returns up to 100 results.'
   ```

3. **Validate Inputs**: Always validate user-provided data
   ```php
   #[Assert\NotBlank]
   #[Assert\Email]
   public string $email
   ```

4. **Handle Errors Gracefully**: Return meaningful error messages
   ```php
   return new CallToolResult(
       content: [new TextContent("Failed: {$error}")],
       isError: true
   );
   ```

5. **Use Type Hints**: Enable better schema generation
   ```php
   public string $name // Good - generates {"type": "string"}
   public $name // Bad - generates broader schema
   ```

6. **Document Side Effects**: Make it clear if a tool modifies data
   ```php
   description: 'Creates a new user account (DESTRUCTIVE - creates database records)'
   ```

7. **Consider Security**: Use API Platform's security features
   ```php
   new McpTool(
       security: "is_granted('ROLE_ADMIN')",
       processor: [AdminTool::class, 'execute']
   )
   ```

## Limitations

* Tools currently support HTTP GET method by default (customizable via `method` parameter)
* Resources are read-only (by design)
* JSON Schema generation requires type hints on class properties
* Circular references in nested objects may require custom serialization groups

## See Also

* [Model Context Protocol Specification](https://modelcontextprotocol.io/)
