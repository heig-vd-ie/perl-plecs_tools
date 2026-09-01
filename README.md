# perl-plecs_tools

This repository contains several projects related to **PLECS**.  There are some tools for working
with simulations, control optimization, and project setup.

> **Important:** This repository contains **multiple independent
> projects in the same GitHub repository**.\
> When working with one of the projects in VS Code, you should open
> **only that project folder** as your VS Code workspace.

## Repository Structure

The repository is organized into several projects:

``` text
perl-plecs_tools/
├── Images/
├── PI_Optimizer/
├── Project_template/
├── Simulation_Runner/
├── Environment_Installation.md
└── README.md
```

* ### PI Optimizer

The **PI Optimizer** is a project for automatically tuning a **PI
controller** using **PLECS**.

The optimizer: - Runs PLECS simulations with different PI controller
parameters. - Evaluates the simulation results using a defined **cost
function**. - Iteratively changes the controller parameters. - Searches
for PI parameters that minimize the selected cost function.

* ### Project Template

The **Project Template** is a starting template for creating new
projects.

It provides a predefined project structure and configuration that can be
reused when starting a new PLECS-related project. The goal is to keep
projects consistent and reduce the amount of setup required for each new
project.

* ### Simulation Runner

The **Simulation Runner** is a project designed to automate **PLECS
simulations**.

It can be used to: - Run PLECS simulations automatically. - Change
simulation/model parameters. - Run multiple simulations with different
parameter values. - Collect and process simulation results.

## Working with the Projects in VS Code

Because the GitHub repository contains **multiple projects**, it is
important to understand how to open them in VS Code.

### Recommended: Open One Project at a Time

If you want to work on the `Simulation_Runner` project, open the
`Simulation_Runner` folder directly in VS Code:

``` text
Simulation_Runner/
├── .cache/
├── .github/
├── .venv/
├── .vscode/
├── data/
├── docs/
├── PLECS_Sim_Runner/
├── src/
├── .envrc
├── .gitignore
├── PLECS_SIM_RUNNER.md
└── requirements.txt
```

In VS Code, use:

**File → Open Folder... → `Simulation_Runner`**

The same principle applies to the other projects:

``` text
File → Open Folder... → PI_Optimizer
```

...


### Do Not Open the Entire Repository for a Single Project

Avoid opening the root of the GitHub repository when you are working on
only one project:

``` text
perl-plecs_tools/
├── PI_Optimizer/
├── Project_template/
├── Simulation_Runner/
└── ...
```

Opening the repository root makes VS Code treat all the projects as part
of the same workspace. This can cause confusion with:

-   Python virtual environments
-   Python interpreters
-   VS Code settings
-   Extensions
-   Project-specific paths
-   `requirements.txt`
-   Source code navigation
-   Environment variables
-   Debugging and launch configurations

Each project should instead be treated as its **own VS Code project**,
even though all projects are stored in the same GitHub repository.

## Why Are Multiple Projects in One Repository?

The projects are related and share a common purpose and development
environment, so they are kept together in one repository.

Think of the repository as a container for several projects rather than
as one single VS Code project.

## Installation

Each project may have its own dependencies and setup instructions.

For environment setup, see:

`Environment_Installation.md`

For a specific project, check its own documentation and
`requirements.txt` file.

## Getting Started

1.  Clone the repository.
2.  Decide which project you want to work on.
3.  Open **only that project's folder** in VS Code.
4.  Create or activate the project's Python environment if required.
5.  Install the project's dependencies.
6.  Follow the project-specific documentation.
7.  Run the PLECS simulations or optimization tools as required.

For example, to work on the Simulation Runner:

``` text
Clone repository
      ↓
Open perl-plecs_tools/Simulation_Runner
      ↓
Select the project's Python environment
      ↓
Install requirements.txt
      ↓
Run the Simulation Runner
```

## Notes

-   PLECS is required for projects that execute PLECS simulations.
-   Make sure the correct Python environment is selected in VS Code for
    the project you are working on.
-   Do not assume that the Python environment or VS Code configuration
    of one project applies to another project.
-   Keep project-specific dependencies inside the corresponding project.