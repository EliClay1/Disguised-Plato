### General Notes
- Track inject deadlines in private tracker
- Keep record of important information, like team identifier
- Immediately make a full copy of the Host, OS, time, interfaces, routes, resolver, listening sockets, running services, installed applications, local users, admins, and active sessions
- Capture firewall/NAT config
- Hash critical configs and write a script to check the hash
- Remove any users that aren't required
- Map each public service to its host, process, service account, configuration, content/data, certificate, dependency, and test method.
### Users
- Create a map of the data as a csv, export the data to my personal system
- Secure accounts 
	- remove ones that aren't required
	- configure different passwords for all the users (myself included)
- Change all application credentials
- check for any form of persistence from red team
	- keys, sudoers, login scripts, profile files, scheduled tasks, services, startup folders, remote management, etc.
### Persistence Checks
- Check any remote access, interactive sessions, etc.
- Check any new or modified users, groups, SSH keys, services, jobs, etc.
- Run a scan every few minutes to check for any unexpected listeners, outbound connections, DNS changes, etc.
- Review any file changes, write paths, etc.
- Treat any change (not done by me) as incident
	- Document > contain > remove > monitor
- Track log data
### Patches
- BACKUP FILES. ALL OF THEM.

### Injects
- Use time blocking to ensure they get submitted, even half written
- Read each one immediately, record the deadline, outputs, etc.
- DO NOT USE AI
