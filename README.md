# Example use cases of Goose + RamaLama in Fedora Linux

This README.md covers several quick and simple examples of potential AI use cases for Goose and RamaLama in Fedora Linux with Qwen 3.5 4B to show what is possible.

**Disclaimer: These instructions are provided for educational purposes.  Review all scripts, configurations, and steps before using or running.  Verify AI outputs independently. Do not process untrusted inputs (such as unknown websites, emails, or files).  Use at your own risk.**

Note: The examples here use a very small Qwen 3.5 4B model that can run with CPU inference, so results could be poor, especially for complex tasks/situations.  If you have access to a GPU, it is highly recommended to use a larger model for much better results.

With these use cases, LLM inference is done locally on the system with RamaLama.

## Prerequisites

* Install packages:  `sudo dnf install -y goose ramalama gnome-terminal python3-pip`

## Start RamaLama

Start Unsloth Qwen 3.5 4B, with vision support.  You might need to tune these settings for your use case; for more details see: https://unsloth.ai/docs/models/qwen3.5

This will download the main model, as well as the `mmproj-F16.gguf` file for vision support.  Depending on your connection speed, it might take a while to download the models:

```
ramalama serve huggingface://unsloth/Qwen3.5-4B-GGUF --runtime-args="--temp 0.6 --top-p 0.95 --min-p 0.00 --top_k 20 --presence_penalty 1.0 --repeat_penalty 1.0 --chat-template-kwargs '{\"enable_thinking\":false}'"
```

Make a note of what port it is listening on.

## Configure Goose

* Open Goose configuration: `goose configure`
* Select `Configure Providers`
* Select `Ollama`
* Set `OLLAMA_HOST` to `127.0.0.1:8080` (replace `8080` with port number RamaLama is listening on if it is a different port number)
* Select the model `unsloth/Qwen3.5-4B-GGUF`
* Run `goose configure` again to disable all extensions
* Select `Toggle Extensions`
* Use `space` to toggle all extensions to `off`, then press `enter`
* Run `goose configure` again to set the permission mode (in this example, I'm using `Approve Mode`)
* Select `goose settings`
* Select `goose mode`
* Select `Approve Mode`

## Use case 1: Interactive Goose session

* Run `goose`
* Type a message and verify you get a response, for example: `What is the Fermi paradox?`
* This is the standard interactive Goose interface where you can easily chat and interact with the LLM model

## Use case 2: Optical character recognition / translation for images

This will add a "Right Click", "Open With" option to process images with optical character recognition using the Qwen model's vision capabilities

* Create `~/.local/share/applications/ocr.desktop` with the following (**note: change <user> to your user name**):
```
[Desktop Entry]
Name=AI - OCR image
Comment=AI - OCR image
Exec=/home/<user>/bin/process_image_with_ai.sh ocr %F
Icon=preferences-system
Terminal=true
Type=Application
MimeType=image/png;image/jpeg;image/jpg;image/webp;image/bmp;image/*
Categories=Utility;Graphics;
NoDisplay=true
```

* Create `bin` directory in home directory: `mkdir ~/bin`
* Create `~/bin/process_image_with_ai.sh` with the following content:
```
#!/bin/bash

OPERATION="$1"
IMAGE_PATH="$2"

if [ ! -f "$IMAGE_PATH" ]; then
    echo "Error: File not found: $IMAGE_PATH"
    exit 1
fi

echo "🖼 Processing image ..."
echo "-----------------------"


if [ "$OPERATION" == "ocr" ]; then
  echo "Make your best attempt to convert this image to text and maintain the text structure. Do not add any other comments, only output the text in the image.  If the text is in a non-English language, translate and output the text in English: $IMAGE_PATH" | goose run -q -i -
fi

if [ "$OPERATION" == "describe" ]; then
  echo "Describe this image in detail:  $IMAGE_PATH " | goose run -q -i -
fi

echo
echo "Press enter to close"
read line
```
* Make script executable:  `chmod 700 ~/bin/process_image_with_ai.sh`
* Run `nautilus -q` to restart Nautilus
* Open Nautilus, right click on image, click `Open With`, and select `AI - OCR image`
* Wait for the model to process the image


## Use case 3: Describe image content

This will add a "Right Click", "Open With" option to process images with Qwen model's vision capabilities to provide a detailed description of the image

* Create `~/.local/share/applications/describe_image.desktop` with the following (**note: change <user> to your user name**):
```
[Desktop Entry]
Name=AI - Describe Image
Comment=AI - Describe Image
Exec=/home/<user>/bin/process_image_with_ai.sh describe %F
Icon=preferences-system
Terminal=true
Type=Application
MimeType=image/png;image/jpeg;image/jpg;image/webp;image/bmp;image/*
Categories=Utility;Graphics;
NoDisplay=true
```
* Run `nautilus -q` to restart Nautilus
* Open Nautilus, right click on image, click `Open With`, and select `AI - Describe Image`
* Wait for the model to process the image

## Use case 4: Goose terminal integration
This configures the Goose terminal integration feature:  https://block.github.io/goose/docs/guides/terminal-integration/

Results will likely be poor with a 4B model for many tasks.  It is highly recommended to use a larger model if you have a GPU available.

* Run `echo 'eval "$(goose term init bash)"' >> ~/.bashrc`
* Run `. .bashrc`
* Goose can now see the commands you've previously run
* Ask questions with `@goose` or `@g` followed by your question
* Example:  `@g "How do I see hidden files with ls?  Output the command only"`

## Use case 5: Convert text to professional/business-friendly text

Example of creating a shortcut that takes whatever you have in the clipboard, and uses Goose/Qwen to convert the text into something more professional.

* Create `~/bin/llm-clipboard.sh` with this content:
```
#!/bin/bash

PROMPT="Rewrite this text with a professional and friendly tone.  Make the message longer if needed.  Output only the rewritten professional text.  Do not provide multiple options, do not provide extra comments, do not suggest a subject line, do not format with markdown"

CLIPBOARD_CONTENT=$(wl-paste)

# Check if clipboard is empty
if [ -z "$CLIPBOARD_CONTENT" ]; then
    echo "❌ Error: Clipboard is empty."
    echo "Please copy some text first (Ctrl+C)."
    echo "Press Enter to exit..."
    read
    exit 1
fi

echo "📋 Processing clipboard content..."
echo "--------------------------------"

# escape double quotes in the content 
CONTENT=$(echo "$CLIPBOARD_CONTENT" | sed 's/"/\\"/g')

TEMP_FILE=$(mktemp /tmp/llm-output-XXXXXX.txt)

echo "$PROMPT:  $CONTENT" | goose run -q -i -  | tee $TEMP_FILE

if [ ! -s $TEMP_FILE ]; then
    echo "❌ Error: No response from LLM."
    echo "Press Enter to exit..."
    read
    rm $TEMP_FILE
    exit 1
fi

# Comment the line below if you don't want the result to replace the original text in the clipboard
cat $TEMP_FILE | wl-copy

rm $TEMP_FILE

echo "Press Enter to close this window..."
read
```
* Make script executable: `chmod 700 ~/bin/llm-clipboard.sh`
* Open `Keyboard` GNOME settings
* Click `View and Customize Shortcuts`
* Click `Custom Shortcuts`
* Click `+` to add new shortcut
* Name shortcut `Convert text to be professional`
* Set command to `gnome-terminal -- /home/<user>/bin/llm-clipboard.sh` **(note: change <user> to your user name)**
* Set a shortcut, i.e. ALT+P
* Copy some text into your clipboard, for example: ***Brian, need an update on status of TPS report!  What is ETA?  We need this ASAP.  Also, don't forget to include a cover report.***
* Press shortcut (ALT+P)
* Wait for the LLM to convert the text to be more professional.  Once completed, the updated text will be copied to the clipboard.

## Use case 6: Use Goose with the Linux MCP server
Configures Goose with the Linux MCP server: https://github.com/rhel-lightspeed/linux-mcp-server

Results will likely be poor with a 4B model.  It is highly recommended to use a larger model if you have a GPU available

* Note: issues were encountered when using Linux MCP server with the latest version of fastmcp, so the previous version was installed before installing linux-mcp-server:  `pip install fastmcp==2.14.5`
* Install the Linux MCP server: `pip install --user linux-mcp-server`
* Run `goose configure`
* Select `Add Extension`
* Select `Command-line Extension`
* Type `linux-mcp-server` for the name
* For command to run, type:  `/home/<user>/.local/bin/linux-mcp-server` **(note: change <user> to your user name)**
* For timeout, specify `300`
* For description, specify `Linux MCP server`
* For `Would you like to add environment variables`, optionally add environment variables to configure the Linux MCP server (refer to Linux MCP server documentation for more details)
* Run `goose` to start a new session
* Ask a question such as `how much free memory is on this system?` 

## License: MIT
Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
