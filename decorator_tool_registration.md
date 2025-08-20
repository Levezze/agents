# Decorator-Based Tool Registration in Python

## Introduction

Decorators are a powerful feature in Python that allow you to modify or enhance functions without changing their core implementation. This document explores how decorators can be used to create a flexible tool registration system.

## What is a Decorator?

A decorator is like a wrapper that adds extra functionality to a function. Think of it as a special label that gives a function additional capabilities.

### Simple Decorator Example

```python
def my_decorator(func):
    def wrapper():
        print("Before function call")
        func()  # Original function
        print("After function call")
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

# Calling say_hello() now does more than just printing
say_hello()
```

## Tool Registry: A Practical Implementation

### Basic Tool Registry

```python
class ToolRegistry:
    # Store of all registered tools
    _tools = {}
    
    @classmethod
    def register(cls, name):
        def decorator(func):
            # Automatically add function to tool collection
            cls._tools[name] = func
            return func
        return decorator
    
    @classmethod
    def get_tool(cls, name):
        # Retrieve a tool by its name
        return cls._tools.get(name)

# Registering tools
@ToolRegistry.register("record_user_details")
def record_user_details(name, email):
    print(f"Recording user: {name}, {email}")
    return {"status": "success"}

@ToolRegistry.register("send_welcome_email")
def send_welcome_email(email):
    print(f"Sending welcome email to {email}")
    return {"status": "email sent"}
```

## Why is This Approach More Flexible?

### 1. Easy Tool Addition

Adding a new tool becomes trivial:

```python
@ToolRegistry.register("generate_report")
def generate_report(user_id):
    # Report generation logic
    pass
```

### 2. Separation of Concerns

- Each tool is defined independently
- Registration mechanism is separate from tool implementation
- Easier to organize and maintain as project grows

### 3. Dynamic Tool Discovery

```python
# List all available tools
print(ToolRegistry._tools.keys())
# Might output: ['record_user_details', 'send_welcome_email', 'generate_report']
```

## Advanced Features: Metadata and More

```python
class ToolRegistry:
    _tools = {}
    _tool_metadata = {}
    
    @classmethod
    def register(cls, name, **metadata):
        def decorator(func):
            # Store function and metadata
            cls._tools[name] = func
            cls._tool_metadata[name] = metadata
            return func
        return decorator
    
    @classmethod
    def get_tool_info(cls, name):
        return {
            'function': cls._tools.get(name),
            'metadata': cls._tool_metadata.get(name, {})
        }

# Usage with metadata
@ToolRegistry.register("complex_tool", 
                       description="A complex tool with additional info",
                       required_permissions=['admin'])
def complex_tool_function():
    pass
```

## Benefits for Larger Projects

- **Scalability**: Add new tools without changing core code
- **Modularity**: Tools can be added from different files/modules
- **Discoverability**: Easy to list and inspect available tools
- **Extensibility**: Can add extra metadata or preprocessing

## Learning Progression

1. Start with simple function registration
2. Add metadata capabilities
3. Implement more advanced features like permission checks, logging, etc.

## Practical Tool Handling Example

```python
def handle_tool_calls(tool_calls):
    results = []
    for tool_call in tool_calls:
        tool_name = tool_call.function.name
        arguments = json.loads(tool_call.function.arguments)
        
        # Dynamically find and call the tool
        tool_function = ToolRegistry.get_tool(tool_name)
        if tool_function:
            result = tool_function(**arguments)
            results.append({
                "role": "tool",
                "content": json.dumps(result),
                "tool_call_id": tool_call.id
            })
    
    return results
```

## Conclusion

Decorator-based tool registration provides a clean, extensible way to manage tools in Python, making your code more modular and easier to maintain.
