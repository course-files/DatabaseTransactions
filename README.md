# Lab: Database Transactions
| Key              | Value                                                                                                                                                                                                                                                                                          |
|:-----------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Course Code**  | MCS 8104, BBT1202, MIT 8107, and DAT 2201                                                                                                                                                                                                                                                      |
| **Course Names** | MCS 8104: Database Management Systems<br>BBT 1202: Advanced Database Systems<br>MIT 8107: Advanced Database Systems<br>DAT 2201: Database Design and SQL                                                                                                                                       |
| **Semester**     | May to August 2026                                                                                                                                                                                                                                                                             |
| **Lecturer**     | Allan Omondi                                                                                                                                                                                                                                                                                   |
| **Contact**      | aomondi@strathmore.edu                                                                                                                                                                                                                                                                         |
| **Note**         | The lecture contains both theory and practice.<br/>This notebook forms part of the practice.<br/>It is intended for educational purposes only.<br/>Recommended citation: [BibTex](https://raw.githubusercontent.com/course-files/DatabaseTransactions/refs/heads/main/RecommendedCitation.bib) |

## Technology Stack

<p align="left">
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/vscode/vscode-original-wordmark.svg" width="40"/>
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/pycharm/pycharm-original.svg" width="40"/>
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/linux/linux-original.svg" width="40" />
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/ubuntu/ubuntu-original-wordmark.svg" width="40" />
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/vim/vim-original.svg" width="40"/>
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/postgresql/postgresql-original-wordmark.svg" width="40"/> 
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/python/python-original.svg" width="40"/>
</p>

## System Architecture

![image](https://github.com/user-attachments/assets/4bd405b6-eeb4-4a50-b47d-9082faccee9a)

## Main Concept

![image](/assets/images/concurrency_problems.png)
![image](/assets/images/isolation_levels.png)

## Repository Structure

```text
.
├── 0_admin_instructions
│   ├── 0_instructions_for_project_setup.md
│   ├── 1_instructions_for_python_installation.md
│   └── 2_instructions_for_project_cleanup.md
├── Docker-Compose.yaml
├── LICENSE
├── README.md
├── RecommendedCitation.bib
├── assets
│   └── images
│       ├── concurrency_problems.png
│       ├── isolation_levels.png
│       └── receipt_sections.png
├── database_transactions.md
├── full_database_transaction_script.sql
├── images
│   ├── apache-httpd-plus-php
│   │   └── Dockerfile
│   ├── mysql
│   │   └── Dockerfile
│   └── ubuntu
│       └── Dockerfile
├── lab_submission_bbt3104
│   └── lab_submission.md
├── lab_submission_dat2201
│   └── lab_submission.md
├── lab_submission_mcs8104
│   └── lab_submission.md
├── lab_submission_mit8107
│   ├── transaction1_CLI.sql
│   ├── transaction2_PHP.php
│   └── transaction2_python.py
├── pos_transaction_demo.py
├── requirements
│   ├── base.txt
│   ├── colab.txt
│   ├── constraints.txt
│   ├── dev.inferred.txt
│   ├── dev.lock.txt
│   ├── dev.txt
│   └── prod.txt
├── requirements.txt
└── script_python
    └── transaction.py

14 directories, 31 files
```

## Setup Instructions

- [Setup Instructions](0_admin_instructions/0_instructions_for_project_setup.md)

## Lab Manual

Refer to the files below for more details:

1. [database_transactions.md](database_transactions.md)
2. [pos_transaction_demo.py](pos_transaction_demo.py)

## Lab Submission Instructions

Refer to the end of the file below for more details:

- [database_transactions.md](database_transactions.md)

## Teardown Instructions (to be done after submitting the lab)

Refer to the file below for more details:

- [Teardown Instructions](0_admin_instructions/2_instructions_for_project_teardown.md)
