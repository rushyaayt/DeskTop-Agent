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



















# 2. *Two*
To give you a genuinely different approach, we will build **Agent #2** using the **ReAct (Reasoning and Acting) Architecture**. 

While the first agent relied on the AI model's native "Tool Calling" JSON API, the ReAct agent uses **Text Parsing**. The AI writes out its internal "Thoughts", decides on an "Action", and then receives an "Observation". 

**Why build this one?**
1. **Model Agnostic:** It works with *any* local text model, even smaller or older ones that don't support native JSON tool-calling.
2. **Transparent:** You can literally read the AI's "Thought" process to see exactly *why* it is doing something.
3. **Keyboard Shortcuts:** It includes native hotkey support (e.g., `Ctrl+C`), which is crucial for desktop automation.

Here is the complete code for the **ReAct Desktop Agent**.

### The ReAct Desktop Agent (Python Backend)

Save this as `react_agent.py`.

```python
import ollama
import pyautogui
import subprocess
import re
import time
import ast

# ==========================================
# 1. SAFETY GUARDRAILS
# ==========================================
DESTRUCTIVE_PATTERNS = [r'rm\s+-rf\s+/', r'format\s+[a-zA-Z]:', r'del\s+/f\s+/s\s+/q']

def is_safe_command(command):
    for pattern in DESTRUCTIVE_PATTERNS:
        if re.search(pattern, command, re.IGNORECASE):
            return False
    return True

# ==========================================
# 2. THE "HANDS" (Available Tools)
# ==========================================
def tool_run_cmd(command: str) -> str:
    if not is_safe_command(command):
        return "Error: Command blocked by safety guardrails."
    try:
        result = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=10)
        return result.stdout.strip() if result.stdout else result.stderr.strip()
    except Exception as e:
        return f"Error: {str(e)}"

def tool_click(x: int, y: int) -> str:
    pyautogui.click(x, y)
    return f"Success: Clicked at ({x}, {y})"

def tool_type_text(text: str) -> str:
    time.sleep(0.5)
    pyautogui.write(text, interval=0.02)
    return f"Success: Typed '{text}'"

def tool_press_key(key_combo: str) -> str:
    # Handles single keys ('enter') or combos ('ctrl', 'c')
    keys = [k.strip().lower() for k in key_combo.split('+')]
    pyautogui.hotkey(*keys)
    return f"Success: Pressed {key_combo}"

# Map string names to actual functions
TOOLS = {
    "run_cmd": tool_run_cmd,
    "click": tool_click,
    "type_text": tool_type_text,
    "press_key": tool_press_key
}

# ==========================================
# 3. THE REACT PROMPT
# ==========================================
SYSTEM_PROMPT = """You are an autonomous AI desktop agent. Your goal is to complete the user's task by interacting with the computer.
You must use the following loop format strictly:

Thought: [Your reasoning about what to do next]
Action: [The tool to use]
Action Input: [The input for the tool, formatted as a python dictionary or string]

Available Tools:
- run_cmd(command: str): Runs a terminal command.
- click(x: int, y: int): Clicks the mouse at screen coordinates.
- type_text(text: str): Types text on the keyboard.
- press_key(key_combo: str): Presses a key or combo (e.g., 'enter', 'ctrl+c', 'alt+f4').

When you have completed the task and have the final answer, use this exact format:
Thought: I now know the final answer.
Final Answer: [Your final response to the user]

Begin!
"""

# ==========================================
# 4. THE REACT EXECUTION LOOP
# ==========================================
def run_react_agent(task: str, model_name: str = "llama3.1"):
    print(f"\n🚀 STARTING REACT AGENT | Task: {task}\n" + "="*50)
    
    # Initialize conversation history
    messages = [
        {'role': 'system', 'content': SYSTEM_PROMPT},
        {'role': 'user', 'content': task}
    ]
    
    max_iterations = 10 # Prevent infinite loops
    iteration = 0

    while iteration < max_iterations:
        iteration += 1
        print(f"\n--- Iteration {iteration} ---")
        
        # 1. ASK THE AI
        response = ollama.chat(model=model_name, messages=messages)
        ai_text = response['message']['content']
        print(f"AI Output:\n{ai_text}\n")
        
        messages.append({'role': 'assistant', 'content': ai_text})

        # 2. CHECK FOR COMPLETION
        if "Final Answer:" in ai_text:
            final_answer = ai_text.split("Final Answer:")[-1].strip()
            print("\n" + "="*50)
            print(f"✅ TASK COMPLETED: {final_answer}")
            return final_answer

        # 3. PARSE ACTION AND ACTION INPUT
        action_match = re.search(r"Action:\s*(.*?)\n", ai_text)
        input_match = re.search(r"Action Input:\s*(.*?)\n", ai_text, re.DOTALL)

        if not action_match or not input_match:
            print("⚠️ AI failed to format Action/Action Input. Halting.")
            break

        action_name = action_match.group(1).strip()
        action_input_raw = input_match.group(1).strip()

        # 4. EXECUTE THE TOOL
        if action_name in TOOLS:
            try:
                # Safely evaluate the input (handles dicts, strings, ints)
                # We wrap it in a safe eval context
                parsed_input = ast.literal_eval(action_input_raw)
                
                if isinstance(parsed_input, dict):
                    result = TOOLS[action_name](**parsed_input)
                else:
                    result = TOOLS[action_name](parsed_input)
                    
                print(f"🛠️ Executed [{action_name}] -> Result: {result}")
                
                # 5. FEED OBSERVATION BACK TO AI
                observation = f"Observation: {result}"
                messages.append({'role': 'user', 'content': observation})
                
            except Exception as e:
                error_msg = f"Observation: Error executing tool. {str(e)}"
                print(f"❌ {error_msg}")
                messages.append({'role': 'user', 'content': error_msg})
        else:
            error_msg = f"Observation: Unknown tool '{action_name}'."
            print(f"❌ {error_msg}")
            messages.append({'role': 'user', 'content': error_msg})

    print("\n⚠️ Reached maximum iterations without finishing.")

# ==========================================
# 5. RUN THE AGENT
# ==========================================
if __name__ == "__main__":
    # Ensure pyautogui doesn't fail on edge cases
    pyautogui.FAILSAFE = True 
    
    # Give the AI 2 seconds to focus the right window before it starts typing/clicking
    print("Starting in 3 seconds. Make sure your desktop is ready...")
    time.sleep(3)

    # Example Task: Open notepad, type a message, and save it using keyboard shortcuts
    user_task = """
    Open notepad. 
    Type the following text: 'ReAct agents are awesome.' 
    Then, use keyboard shortcuts to save the file to the desktop as 'react_test.txt'. 
    (Hint: Use Ctrl+S, then type the filename, then press Enter).
    """
    
    run_react_agent(user_task)
```

### How to use this Agent:

1. **Install Dependencies:**
   ```bash
   pip install ollama pyautogui
   ```
2. **Pull a Model:** 
   ReAct parsing works best with models that follow instructions well. Llama 3 or Mistral are great.
   ```bash
   ollama pull llama3.1
   ```
3. **Run the Script:**
   ```bash
   python react_agent.py
   ```

### Why this architecture is powerful:
If the AI makes a mistake (e.g., it tries to click the wrong coordinate), it will see the `Observation` fail. In the next `Thought` block, it will realize its mistake and say: *"Observation showed the click failed. I will try using the `run_cmd` tool to open the app via terminal instead."* This self-correction loop is the hallmark of advanced autonomous agents.
