✅ Official Workflow From Now On

🔹 Daily Work (Dev Work)

You NEVER work on main.

You work like this:

develop  ← base branch
   ↑
feature/something


⸻

🔧 When Starting New Work

From inside repo:

git checkout develop
git pull origin develop
git checkout -b feature/fix-asset-paths

Work → commit → push:

git add .
git commit -m "Fix asset path handling"
git push -u origin feature/fix-asset-paths

Then:

Merge into develop:

git checkout develop
git merge feature/fix-asset-paths
git push

Deploy DEV from develop.

⸻

🚀 When Ready For Production

From develop:

git checkout main
git merge develop
git push

Then tag release:

git tag release-2026-02-19
git push origin release-2026-02-19

Then deploy production.

⸻

🔒 Optional (Recommended in GitHub UI)

Go to each repo → Settings → Branches

Protect main:
	•	Require pull request before merge
	•	Prevent direct push

Since you’re solo, this is optional but good discipline.

⸻

🔵 Summary

From now on:
	•	develop = dev environment
	•	main = production
	•	feature/* = work branches

⸻
