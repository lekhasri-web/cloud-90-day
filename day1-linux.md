Built: created linux-lab in /root, created subfolders docs and backup using mkdir,
       created files a.txt and b.txt inside docs using touch,
       verified using ls, copied file a.txt from docs to backup, 
       moved and renamed file b.txt from docs to notes.txt in backup folder. 
       Verified structure using ls -lah. Removed folder linux-lab using rm -ri where -r goes through nested files ,
       -i will ask in interactive mode before deleting as they will be deleted permanently. Used man mkdir to learn about -p.
       
Broke: I typed mkdir docs backup /linux-lab. I expected docs and backup to be created inside linux-lab, but the leading / created a separate linux-lab at the top of the filesystem, and docs and backup landed in /root instead.

Fixed: I ran ls and pwd to see where things actually were, deleted the extra folders with rmdir, then recreated the subfolders with mkdir linux-lab/docs linux-lab/backup, with no / at the start of the path.
