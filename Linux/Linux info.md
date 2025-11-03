  **fping** uses Internet Control Message Protocol (ICMP) requests to determine if a host is live or not. However, with **fping**, we can specify any number of targets, including a subnet, making it more versatile than the **ping** command. Instead of sending a packet to one target until it replies or times out, **fping** will move to the next target after each request.
> fping -agq 10.211.11.0/24

Linux system logs are stored in the `/var/log/` folder in plain text.
Главная техника поиска логов - просмотр файла и grep по нужным словам. Также главным способом нахождения вредоноса - дерево процессов по ppid.

Search for potential logins across all logs (/var/log):
`grep -R -E "auth|login|session" /var/log`

The first and often the most useful log file you want to monitor is `/var/log/auth.log`. It contains authentication events, it can also store user management events, launched sudo commands, and much more. 
`cat /var/log/auth.log | grep -E 'session opened|session closed'`

![[Снимок экрана 2025-10-01 в 21.06.39.png]]

 Each successful logon and logoff is logged, and you can see them by filtering the events containing the "session opened" or "session closed" keywords.
 `cat /var/log/auth.log | grep "sshd" | grep -E 'Accepted|Failed'`

You can also use the same log file to detect user management events. This is easy if you know basic Linux commands: If useradd is a command to add new users, just look for a "useradd" keyword to see user creation events!
`cat /var/log/auth.log | grep -E '(passwd|useradd|usermod|userdel)\['`

You may encounter interesting or unexpected events. For example, you may find commands launched with sudo, which can help track malicious actions.
`cat /var/log/auth.log | grep -E 'COMMAND='`

`/var/log/kern.log`: Kernel messages and errors, useful for more advanced investigations
 `/var/log/syslog (or /var/log/messages)`: A consolidated stream of various Linux events
 `/var/log/dpkg.log (or /var/log/apt)`: Package manager logs on Debian-based systems

You might also monitor a specific program, and to do this effectively, you need to use its logs. For example, analyze database logs to see which queries were run, mail logs to investigate phishing, container logs to catch anomalies, and web server logs to know which pages were opened, when, and by whom .An example from the typical Nginx web server logs:
`cat /var/log/nginx/access.log`

Command to view bash history:
`cat /home/ubuntu/.bash_history`

There are [over 300](https://man7.org/linux/man-pages/man2/syscalls.2.html) system calls in Linux, like `execve` to execute a program. Since there is nearly no way for attackers to bypass system calls, all you have to do is choose the system calls you'd like to log and monitor. In the next task, you will try it in practice using auditd. ![[Снимок экрана 2025-10-02 в 14.36.11.png]]

 Key Takeaways
- Linux logging can be chaotic, but it often stores enough details to detect the threat
- Logs are kept in `/var/log/` folder by default and are usually stored in plain text
- The top three log sources for SOC are auth.log, app-specific logs, and runtime logs
- Bash history is unreliable for SOC; better use auditd or an alternative solution

Например, искать события ssh в auth.log можно по словам "ssh" и "Failed"/"Accepted", чтобы найти подозрительную активность.

you can:
- Use **web logs** to detect a variety of web attacks
- Use **database logs** to detect suspicious SQL queries
- Use **VPN logs** to detect abnormal VPN login events

![[Снимок экрана 2025-10-02 в 18.04.09.png]]

![[Снимок экрана 2025-10-02 в 18.51.53.png]]

![[Снимок экрана 2025-10-02 в 18.53.50.png]]

![[Снимок экрана 2025-10-02 в 19.05.21.png]]

In Linux, a _process_ is a running instance of a program.

lsof is a utility that lists information about files opened by processes on a system.

`pstree` is a command-line utility that displays processes visually as a tree, showing the parent-child relationships between processes. This utility can help identify the origin of the suspicious processes and understand their relationship to other processes in the system.
> pstree -p -s *pid*

Cronjobs are scheduled tasks executed automatically at predefined intervals by the **cron daemon**. The cron daemon is a background process responsible for managing cronjobs based on configuration files known as crontabs. Users can have their crontab file stored in the `/var/spool/cron/crontabs` directory. The main crontab file at `/etc/crontab` governs system-wide cronjobs.

- **/etc/cron.hourly/** - System cronjobs that run once per hour.
- **/etc/cron.daily/** - System cronjobs that run once per day.
- **/etc/cron.weekly/** - System cronjobs that run once per week.
- **/etc/cron.monthly/** - System cronjobs that run once per month.
- **/etc/cron.d/** - Additional custom system cronjobs.

`sudo crontab -l -u *username*`

Cron logs record the execution of scheduled tasks managed by the cron daemon and provide a chronological record of when cron jobs were executed, along with any associated output or error messages. On Debian-based systems, cron execution logs are typically stored in `/var/log/syslog`. 

https://github.com/DominicBreuker/pspy - утилита для просмотра запускаемых крон.

In Linux, services refer to various background processes or daemons that run continuously, performing tasks such as managing system resources, providing network services, or handling user requests. For example, the cron daemon we analysed previously ran the cronjobs. Other common services include SSH (sshd) for secure shell or the Apache HTTP Server (httpd). Typically, services are configured using the system's service management utility - **systemd** or init

утилита journalctl

**/etc/init.d/** - This directory typically contains scripts for system service management in traditional SysV init systems and is primarily responsible for starting, stopping, restarting, and managing system services. Each script corresponds to a specific service and follows a standardised naming convention. For example, you might find scripts like `apache2`, `mysql`, or `ssh`, which control the Apache web server, MySQL database server, and OpenSSH server, respectively. These scripts are usually written in Bash or another shell scripting language.

**/etc/systemd/system/** - This directory is related to the systemd init system, which has become the default init system in many modern Linux distributions. As discussed in the previous task, services are defined by unit files stored in this directory. They specify various parameters and actions related to a service, such as its dependencies, startup behaviour, environment variables, and more.

Many desktop environments place autostart scripts in the user's home directory. As such, they can be a hiding spot for user-targeted malware. Accessing other users' home directories typically requires elevated privileges, such as using `sudo` or `su` commands.

**~/.config/autostart/** - The autostart scripts' syntax is usually in the form of `.desktop` files, which are plain text files with a specific format. For example, suppose a user wants to automatically launch a custom script that sets up their development environment upon logging into their Linux desktop. They've written a shell script named `setup_dev_env.sh`, located in their home directory
![[Снимок экрана 2025-10-03 в 18.22.17.png]]

For example, **Firefox** organises user data within profile directories, often found in `~/.mozilla/firefox/`. **Google Chrome** typically stores user profiles (history, web data, login databases, etc.) in `~/.config/google-chrome/`.

> `sudo find /*dir* -type *type_of_file* \( -path "*/name_of_file" \) 2>/dev/null` - поиск файлов по имени и вывод директорий.

Самые распространённые точки входа для атак:
- Открытые порты
- Запущенные сервисы с уязвимостями
- Сетевое взаимодействие

Some of the key points where we could find the incident traces are highlighted below:
- System logs
- auth.log, syslog, krnl.log, etc
- Network traffic
- Running processes
- Running services
- The integrity of the files and processes

Processes and network communication are crucial in any operating system in incident investigations. Monitoring and analyzing processes, especially those with network communication, can help identify and address security incidents.

Osq:
`SELECT pid, fd, socket, local_address, remote_address, remote_port FROM process_open_sockets WHERE local_address="10.10.124.33";`

Malware often tries to keep a footprint in the system such that it keeps running even after a system restart. This is called persistence. For example, If a malware adds itself to the startup registry keys, it will persist even after a system restart.

/etc/passwd contains:
- Username.
- The password placeholder is represented by x or *, indicating that the password is stored in the/etc/shadow file.
- User ID assigned to the user
- Group ID assigned to the user.
- User's home directory.
- Path to user's default shell.

All services installed and enabled on the Linux host can be found in the `/etc/systemd/system` directory.

journalctl is a command-line utility in Linux for querying and displaying logs from the systemd journal.

dpkg.log - file when dpkg activity collects

- Location: `/var/log/syslog`
- This is useful for identifying system-wide events, errors, and warnings. Can provide insights into issues with system components or services.
- It contains general system messages, including kernel messages, system services, and application logs.
- This log file is useful for identifying system-wide events, errors, and warnings.
- It can provide insights into issues with system components or services.

- Location: `/var/log/messages`
- Similar to `syslog`, this file includes system messages and kernel logs.
- Useful for diagnosing system issues and tracking system activity.
- Finding unusual entries related to hardware or kernel errors might signal an attempt to tamper with system components.
- For example, repeated kernel panic messages could indicate a denial-of-service attack targeting system stability.

- Location: `/var/log/auth.log`
- This file logs authentication attempts, including successful and failed login attempts.
- It's an important log file for detecting unauthorized access attempts and brute-force attacks.
- For example, finding multiple failed login attempts from an unfamiliar IP address or unusual login times might indicate a brute-force attack or an attempt to gain unauthorized access.

Some of the key examples of the events that can be classified as incidents are:

- Failed login attempts.
- Successful login attempt but at the odd time (After Office Hours or on weekends -> depending on the context of the company).
- Suspicious network communication.
- System errors.
- User account creation on the sensitive server.
- Outbound traffic is initiated from the web server.

Attackers often target directories with write permissions to upload malicious files. Common writable directories include:
- **/tmp**: The temporary directory is writable by all users, making it a common choice.
- **/var/tmp**: Another temporary directory commonly with world write permissions.
- **/dev/shm**: The shared memory file system, which is also normally writable by all users.

|                                       |                                                                           |
| ------------------------------------- | ------------------------------------------------------------------------- |
| `find / -group GROUPNAME 2>/dev/null` | Retrieve a list of files and directories owned by a specific group.       |
| `find / -perm -o+w 2>/dev/null`       | Retrieve a list of all world-writable files and directories.              |
| `find / -type f -cmin -5 2>/dev/null` | Retrieve a list of files created or changed within the last five minutes. |

_Exiftool_ is a Perl-based command-line utility with extensive capabilities for extracting and altering metadata from files by parsing their headers and embedded metadata structures.

md5sum and sha256 - команды для построения хэша.

In Unix-based systems, three main timestamps are commonly recorded:

- **Modify Timestamp (mtime):** This timestamp reflects the last time the **contents** of a file were modified or altered. Whenever a file is written to or changed, its _mtime_ is updated.
- **Change Timestamp (ctime):** This timestamp indicates the last time a file's **metadata** was changed. Metadata includes attributes like permissions, ownership, or the filename itself. Whenever any metadata associated with a file changes, its _ctime_ is updated.
- **Access Timestamp (atime):** This timestamp indicates the last time a file was **accessed** or read. Whenever a file is opened, its _atime_ is updated.

`groups investigator` - To determine which groups a specific user is a member of, we can run the following command
`getent group adm` - Alternatively, to list all of the members of a specific group, we can run the following command

`last` and `lastlog`

The `who` command is a very straightforward command that can be used to display the users that are currently logged into the system.

`stat` - command to get metadata about file

If any files have been modified or corrupted, `debsums` will report them, citing potential issues with the package's integrity. This can be useful in detecting malicious modifications and integrity issues within the system's packages.