# Clean up unused Docker objects automatically 
##  Edit the Crontab
- run `crontab -e` 
- add the command `0 0 * * * /usr/bin/docker system prune -f >> /var/log/docker-prune.log 2>&1` 


## command explanation 
The command is a cron job that schedules a task to automatically run at a specific time.
----------------------------------------
## Timing 

 `(0 0 * * *)` defines when the command should run:

- 0 (minute): Run at minute 0.
- 0 (hour): Run at hour 0 (i.e., midnight).
- `*` (day of month): Every day of the month.
- `*` (month): Every month.
- `*` (day of week): Every day of the week.
##### this means that the job will run every day at midnight.


## Clean up unused Docker objects
- `/usr/bin/docker` This is the full path to the Docker executable.
- `system prune` This Docker command removes unused data, including stopped containers, dangling images, unused networks, and unused volumes.
- `-f` (force flag): This option forces the prune operation without prompting for confirmation.
## Logging Output: >> /var/log/docker-prune.log 2>&1
- `>> /var/log/docker-prune.log` This appends the output of the command (both standard output and errors) to the file /var/log/docker-prune.log.

- `2>&1` This redirects error output (stderr) to the same place as the standard output (stdout). This ensures that both regular output and errors go into the same log file.
## Summary
- This cron job will run every day at midnight, executing the Docker system prune command to clean up unused Docker objects. It forces the cleanup `(-f)` and logs both the standard output and any errors to `/var/log/docker-prune.log`
- list the current cron jobs to verify that the entry has been added by running `crontab -l`
