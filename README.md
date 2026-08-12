**After repository creation:**
- [ ] Update this `README.md`. Update the Project Name, description, and all sections. Remove this checklist.
- [ ] **Specify your license.** This template does NOT ship with a default license. Identify your project's approved license per your organization's license approval guidelines. 
- [ ] **Create `LICENSE.txt`.** Replace the placeholder text in `LICENSE.txt` with the full text of your project's approved license.
- [ ] **Update the License section** below to name your approved license and link to it.
- [ ] Search this repo for "REPLACE-ME" and update all instances accordingly
- [ ] Update `CONTRIBUTING.md` as needed
- [ ] Review the workflows in `.github/workflows`, updating as needed. See https://docs.github.com/en/actions for information on what these files do and how they work.
- [ ] Review and update the suggested Issue and PR templates as needed in `.github/ISSUE_TEMPLATE` and `.github/PULL_REQUEST_TEMPLATE`
- [ ] Remove this checklist

# Qualcomm SKILLS

Agent Skills for Qualcomm products. These skills can be installed into coding agents to run Qualcomm Product specific workflows.
These skills can also be self discovered by Qualcomm Developer Tools.

## Branches

**main**: Primary development branch. Contributors should develop submissions based on this branch, and submit pull requests to this branch.

## Requirements

List requirements to run the project, how to install them, instructions to use docker container, etc...

## Installation Instructions
Two Options : 
1. Find the skills relevant to your product, OS combination and install the needed skills in your coding agent. Follow the instructions of your coding agent (e.g. for Claude, copy the skills in .skills directory).
2. Create a skills.json file in your Skills root with appropriate events. This will enable Qualcomm Developer tools to discover these skills. The Skills.json file needs to be aligned with Qualcomm Developer Tools format. See the skills.json in some pre-existing skills on this repo.

## Usage
The skills in this repo should be used with the Agentic tools.

## Development

How to develop new features/fixes for the software. Maybe different than "usage". Also provide details on how to contribute via a [CONTRIBUTING.md file](CONTRIBUTING.md).

## Getting in Contact

* [Report an Issue on GitHub](../../issues)
* [Open a Discussion on GitHub](../../discussions)

## License

*skills* is licensed under the [BSD-3-Clause-Clear License](https://spdx.org/licenses/BSD-3-Clause-Clear.html). See [LICENSE.txt](LICENSE.txt) for the full license text.
