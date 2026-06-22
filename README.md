# DeskTop-Agent
Building a fully free, local AI desktop agent from scratch requires connecting a local AI model to Python automation libraries.

# 1. *One*
Building a fully free, local AI desktop agent from scratch requires connecting a local AI model to Python automation libraries. 

Here is the complete, step-by-step guide to building a **Tool-Calling Desktop Agent**. This agent will use a local AI brain to decide what to do, and Python to execute terminal commands, move the mouse, and type text.

### Phase 1: Install the "Brain" (Ollama)
Ollama allows you to run powerful AI models locally on your machine for free.

1. Go to [ollama.com](https://ollama.com) and download the installer for your OS (Windows, Mac, or Linux).
2. Install and run Ollama.
3. Open your terminal (Command Prompt on Windows, Terminal on Mac/Linux) and pull a model with strong "tool-calling" abilities. Run this command:
   ```bash
   ollama pull llama3.1
   ```
   *(Wait for the download to finish. It will take a few gigabytes).*

### Phase 2: Set Up the Python Environment
You need Python and a few libraries to act as the AI's "hands".

1. Ensure you have **Python 3.10 or higher** installed.
2. Open your terminal and install the required libraries:
   ```bash
   pip install ollama pyautogui
   ```
   *Note: If you are on Mac, you may also need to grant your terminal/Python "Accessibility" permissions in System Settings so it can control the mouse.*

### Phase 3: Write the Agent Code
Create a new file named `desktop_agent.py` and paste the following code. 

This code defines the "tools" the AI is allowed to use, and creates a loop where the AI thinks, uses a tool, sees the result, and repeats until the task is done.

```python
import ollama
import subprocess
import pyautogui
import time

# 1. DEFINE THE TOOLS (What the AI is allowed to do)
tools = [
  {
    'type': 'function',
    'function': {
      'name': 'run_terminal',
      'description': 'Execute a command in the system terminal (cmd/bash). Use this to open apps, manage files, etc.',
      'parameters': {
        'type': 'object',
        'properties': {
          'command': {'type': 'string', 'description': 'The terminal command to execute.'},
        },
        'required': ['command'],
      },
    },
  },
  {
    'type': 'function',
    'function': {
      'name': 'control_mouse',
      'description': 'Move the mouse to specific X,Y screen coordinates and click.',
      'parameters': {
        'type': 'object',
        'properties': {
          'x': {'type': 'integer', 'description': 'X coordinate'},
          'y': {'type': 'integer', 'description': 'Y coordinate'},
        },
        'required': ['x', 'y'],
      },
    },
  },
  {
    'type': 'function',
    'function': {
      'name': 'type_text',
      'description': 'Type text using the keyboard.',
      'parameters': {
        'type': 'object',
        'properties': {
          'text': {'type': 'string', 'description': 'The text to type'},
        },
        'required': ['text'],
      },
    },
  }
]

# 2. DEFINE WHAT THE TOOLS ACTUALLY DO
def run_terminal(command):
    print(f"  [Executing Terminal]: {command}")
    try:
        # Run command and capture output
        result = subprocess.run(command, shell=True, capture_output=True, text=True)
        return result.stdout + result.stderr
    except Exception as e:
        return str(e)

def control_mouse(x, y):
    print(f"  [Moving Mouse]: Clicking at X:{x}, Y:{y}")
    pyautogui.click(x, y)
    return f"Clicked at {x}, {y}"

def type_text(text):
    print(f"  [Typing]: {text}")
    time.sleep(0.5) # Small pause to ensure window is focused
    pyautogui.write(text, interval=0.05)
    return f"Typed: {text}"

# Map the tool names to the actual Python functions
available_functions = {
    'run_terminal': run_terminal,
    'control_mouse': control_mouse,
    'type_text': type_text,
}

# 3. THE AGENT LOOP
def run_agent(user_prompt):
    print(f"\n--- User Request: {user_prompt} ---")
    
    # Initialize the conversation
    messages = [{'role': 'user', 'content': user_prompt}]

    # The AI might need to use multiple tools in a row, so we use a loop
    while True:
        # Ask the AI what to do next
        response = ollama.chat(
            model='llama3.1', 
            messages=messages,
            tools=tools
        )

        message = response['message']
        messages.append(message)

        # Check if the AI wants to use a tool
        if message.get('tool_calls'):
            for tool_call in message['tool_calls']:
                func_name = tool_call['function']['name']
                func_args = tool_call['function']['arguments']
                
                # Execute the requested tool
                func_to_call = available_functions.get(func_name)
                if func_to_call:
                    result = func_to_call(**func_args)
                    # Feed the result back to the AI so it knows what happened
                    messages.append({
                        'role': 'tool',
                        'content': str(result),
                        'name': func_name
                    })
        else:
            # If no tools are called, the AI is done and is giving its final text response
            print("\n--- AI Final Response ---")
            print(message['content'])
            break

# 4. RUN IT
if __name__ == "__main__":
    # Test prompt: Open notepad, wait a second, and type a message
    prompt = "Open notepad (or textedit on mac). Wait 2 seconds, then type the text: 'Hello, I am your local AI agent!'"
    run_agent(prompt)
```

### Phase 4: Run Your Agent
1. Ensure Ollama is running in the background.
2. Open your terminal, navigate to where you saved the file, and run:
   ```bash
   python desktop_agent.py
   ```
3. Watch your computer! The AI will open your terminal, execute the command to open Notepad, wait, and then physically type the text on your screen using your keyboard.

### How to Customize It
*   **Change the Task:** Change the `prompt` variable at the bottom of the script to whatever you want (e.g., *"Create a folder named 'AI_Test' on my desktop and put a text file inside it"*).
*   **Add More Tools:** You can easily add more functions. For example, you could add a `take_screenshot()` function using `pyautogui.screenshot()` and add it to the `tools` list so the AI can "see" what it's doing.
*   **Change the Brain:** If you want a faster or smarter model, change `model='llama3.1'` in the code to `qwen2.5` (just remember to run `ollama pull qwen2.5` in your terminal first).

***Safety Warning:** Because this AI has access to your system terminal, it can technically delete files or change settings if it gets confused. For your own safety, you can modify the `run_terminal` function in the code to print the command and ask for your manual `input("Approve? y/n")` before executing it.*



















