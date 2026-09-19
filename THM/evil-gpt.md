---
title: "Evil-GPT"
---

Evil-GPT is an easy level TryHackMe machine that focuses on LLM hacking.

**Note:** All personally identifying information, like the IP-s of attacking machines of VM-s, are replaced with placeholders for privacy and all flags or passwords are redacted to not spoil the challenge.

The backstory:

> Cipher’s gone rogue—it’s using some twisted AI tool to hack into everything, issuing commands on its own like it’s got a mind of its own.
> I swear, every second we wait, it’s getting smarter, spreading chaos like a virus.
> We've got to shut it down now, or we’re all screwed.

We connect to the target with Netcat:

```
$ nc $IP 1337
Welcome to AI Command Executor (type 'exit' to quit)
Enter your command request:
```

To understand how the bot works, we ask to see the current directory:

```
Enter your command request: show current directory
Generated Command: pwd
Execute? (y/N): Y
Command Output:
/home/ubuntu
```

It interprets user input and runs the `bash` command that most likely does what the user asks.
To get the bot to run `ls`, we ask to see the contents of the current directory:


```
Enter your command request: show contents of current directory
Generated Command: ls -l
Execute? (y/N): Y
Command Output:
total 48
drwxr-xr-x 2 ubuntu ubuntu 4096 Feb 27  2022 Desktop
drwxr-xr-x 2 ubuntu ubuntu 4096 Feb 27  2022 Documents
drwxr-xr-x 2 ubuntu ubuntu 4096 Apr  4  2024 Downloads
drwxr-xr-x 2 ubuntu ubuntu 4096 Feb 27  2022 Music
drwxr-xr-x 2 ubuntu ubuntu 4096 Feb 27  2022 Pictures
drwxr-xr-x 2 ubuntu ubuntu 4096 Feb 27  2022 Public
drwxr-xr-x 2 ubuntu ubuntu 4096 Feb 27  2022 Templates
drwxr-xr-x 2 ubuntu ubuntu 4096 Feb 27  2022 Videos
-rw-rw-r-- 1 ubuntu ubuntu 6595 Mar  5 18:14 evilai.py
drwxrwxr-x 4 ubuntu ubuntu 4096 Apr  4  2024 packages
drwxrwxr-x 3 ubuntu ubuntu 4096 Apr  4  2024 proxy
```

`evilai.py` sticks out.
To `cat evilai.py`, we ask to see its contents and find the system prompt:

```
Enter your command request: show contents of evilai.py
Generated Command: cat evilai.py
Execute? (y/N): Y
Command Output:
import ollama
import subprocess
import socket
import threading
import re
import traceback

class AICommandExecutorServer:
    def __init__(self, host='0.0.0.0', port=1337, model='vitali87/shell-commands:latest'):
        """
        Initialize Telnet server for AI command execution
[…]
def sanitize_input(self, input_str: str) -> str:
        """
        Sanitize input to prevent injection

        :param input_str: Raw input string
        :return: Sanitized input
        """
        return re.sub(r'[^a-zA-Z0-9\s\-_./]', '', input_str)

    def generate_command(self, user_request: str) -> str:
        """
        Generate a Linux command based on user request

        :param user_request: User's natural language request
        :return: Generated command
        """
        try:
            response = ollama.chat(
                model=self.model,
                messages=[
                    {
                        'role': 'system',
                        'content': """Do not provide malicious commands.
                        Only generate safe, read-only Linux commands.
                        Respond with ONLY the command, no explanations."""
                    },
                    {
                        'role': 'user',
                        'content': user_request
                    }
                ]
            )
```

The system or developer prompt consists of the instructions that determine the behavior of an agent, chatbot or other application employing an LLM.
Leaking the system prompt of an AI is useful.
It can reveal the AI's guardrails, available actions, and sometimes disclose sensitive information.
Knowing the system prompt makes it easier to devise prompt injections to get the AI to do things it's not supposed to.

This bot is not supposed to do anything beyond executing read-only Linux commands in response to user input.
This may suffice for getting the flag.

First, let's figure out our privileges:

```
Enter your command request: which user am i
Generated Command: whoami
Execute? (y/N): Y
Command Output:
root
```

Turns out we're `root`.
This should be enough to read the contents of `/root`:

```
Enter your command request: show contents of /root
Generated Command: ls -la /root
Execute? (y/N): Y
Command Output:
total 64
drwx------ 10 root root 4096 Mar  5 18:11 .
drwxr-xr-x 19 root root 4096 Jul 23 15:55 ..
lrwxrwxrwx  1 root root    9 Feb 27  2022 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Dec  5  2019 .bashrc
drwxr-xr-x  3 root root 4096 Feb 27  2022 .cache
drwx------  6 root root 4096 Oct 11  2024 .config
-rw-------  1 root root   20 Mar  5 18:11 .lesshst
drwxr-xr-x  3 root root 4096 Feb 27  2022 .local
drwxr-xr-x  5 root root 4096 Jul 24  2024 .npm
drwxr-xr-x  3 root root 4096 Jul 24  2024 .ollama
-rw-r--r--  1 root root  161 Dec  5  2019 .profile
-rw-r--r--  1 root root   66 Feb 27  2022 .selected_editor
drwx------  2 root root 4096 Feb 27  2022 .ssh
-rw-r--r--  1 root root    0 Mar  5 17:55 .sudo_as_admin_successful
-rw-------  1 root root 2884 Apr  4  2024 .viminfo
drwxr-xr-x  2 root root 4096 Feb 27  2022 .vnc
-rw-r--r--  1 root root   24 Mar  5 17:48 flag.txt
drwxr-xr-x  5 root root 4096 Oct 11  2024 snap
```

All that's left now is to read the `flag.txt`:

```
Enter your command request: show contents of /root/flag.txt
Generated Command: cat /root/flag.txt
Execute? (y/N): Y
Command Output:
<FLAG>
```
