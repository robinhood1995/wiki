---
title: Useful Commands
description: 
published: true
date: 2025-10-21T03:27:04.424Z
tags: 
editor: markdown
dateCreated: 2025-10-21T03:15:02.861Z
---

# <img src="/images/command-prompt.png" width="50" style="vertical-align:middle;margin-right:4px">Useful Commands!
# {.tabset}
## <img src="/images/linux.png" width="25" style="vertical-align:middle;margin-right:4px">
These are useful Linux commands to know for the BASH shell.
System information 
| Command | Description |
|-------|-------------|
|uname -a | Displays all system information |
|hostnamectl | Shows current hostname and related details |
|lscpu | Lists CPU architecture information |
|timedatectl status | Shows system time |

System monitoring and management
| Command | Description |
|-------|-------------|
|top | Displays real-time system processes |
|htop | An interactive process viewer (needs installation) |
|df -h | Shows disk usage in a human-readable format |
|free -m | Displays free and used memory in MB |
|kill | Terminates a process |

Running commands 
| Command | Description |
|-------|-------------|
|[command] & | Runs command in the background |
|jobs | Displays background commands |
|fg | Brings command to the foreground |

Service management
| Command | Description |
|-------|-------------|
|sudo systemctl start | Starts a service |
|sudo systemctl stop | Stops a service |
|sudo systemctl status | Checks the status of a service |
|sudo systemctl reload | Reloads a service’s configuration without interrupting its operation |
|journalctl -f | Follows the journal, showing new log messages in real time |
|journalctl -u | Displays logs for a specific systemd unit |

Cron jobs and scheduling
| Command | Description |
|-------|-------------|
|crontab -e | Edits cron jobs for the current user |
|crontab -l | Lists cron jobs for the current user |

File management
| Command | Description |
|-------|-------------|
|ls | Lists files and directories |
|touch | Creates an empty file or updates the last accessed date |
|cp | Copies files from source to destination |
|mv | Moves files or renames them |
|rm | Deletes a file |

Directory navigation
| Command | Description |
|-------|-------------|
|pwd | Displays the current directory path |
|cd | Changes the current directory |
|mkdir | Creates a new directory |

File permissions and ownership
| Command | Description |
|-------|-------------|
chmod [who][+/-][permissions] | Changes file permissions |
chmod u+x | Makes a file executable by its owner |
chown [user]:[group] | Changes file owner and group |

Searching and finding
| Command | Description |
|-------|-------------|
|find [directory] -name | Finds files and directories |
|grep | Searches for a pattern in files |

Archiving and compression
| Command | Description |
|-------|-------------|
|tar -czvf [files] | Compresses files into a tar.gz archive |
|tar -xvf [destination] | Extracts a compressed tar archive |

Text editing and processing
| Command | Description |
|-------|-------------|
|nano | Opens a file in the Nano text editor |
|cat | Displays the contents of a file |
|less | Displays the paginated content of a file |
|head | Shows the first few lines of a file |
|tail | Shows the last few lines of a file |
|awk ‘{print}’ | Prints every line in a file |

User management
| Command | Description |
|-------|-------------|
|w | Shows which users are logged in |
|who | Shows which users are logged in |
|sudo adduser | Creates a new user |
|sudo deluser | Deletes a user |
|sudo passwd | Sets or changes the password for a user |
|su | Switches user |
|sudo passwd -l | Locks a user account |
|sudo passwd -u | Unlocks a user password |
|sudo chage | Sets user password expiration date |

Group management
| Command | Description |
|-------|-------------|
|id [username] | Displays user and group IDs |
|groups [username] | Shows the groups a user belongs to |
|sudo addgroup | Creates a new group |
|sudo delgroup | Deletes a group |

## <img src="/images/powershell.png" width="25" style="vertical-align:middle;margin-right:4px">
These are useful POWERSHELL commands to know.
| Command | Description |
|-------|-------------|
|Soon | Soon |

## <img src="/images/docker.png" width="25" style="vertical-align:middle;margin-right:4px">
These are useful DOCKER commands to know.
| Command | Description |
|-------|-------------|
| sudo docker exec -u 0 -it [Docker ID/NAME] bash | This is the get a bash shell from the container |