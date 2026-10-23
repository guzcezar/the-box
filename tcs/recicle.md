git checkout --orphan temp_branch && \
git add -A && \
git commit -m "What's in the box?" && \
git branch -D main && \
git branch -m main && \
git push --force-with-lease origin main && \
git reflog expire --expire=now --all && \
git gc --prune=now -q && \
git log --oneline

