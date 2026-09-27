# SyncSnitch cloud runner

GitHub Actions runner for the SyncSnitch demo site (code: https://github.com/Kshiti26-11/Demo).
The site dispatches `syncsnitch-agents.yml`; each job runs the three agents and pushes the live state of the run
to the branch `syncsnitch-live/<run_id>`, which the site shows. Set up by `bash scripts/cloud_setup.sh`.
