# Create an Openshift learning lab

The goal of this project is to work towards openshift administrator certification. While virtualization is important, this will focus on the container platform. 

## Stack

- AI Agents: IBM Bob and Cursor
- Harness: orca (https://onorca.dev)
- Bob Software Factory: https://ibm.github.io/bob-software-factory
- Single node openshift running as a VM on proxmox 
- `mise` as local task runner and environment manager


## Guide

I recently presented around terraform and created a lab guide that allowed users to click a button to show they had completed a step and it would update a diagram on the screen showing that it had been done. The guide is at https://images.gh40-dev.systems/corrected-lab-1952.html
I want to mimic this behavoir with this new guide with the added twist that I want the system we are building to be able to check on that progress. So a user gets a box showing them how to create a new project, they click `I did it` or something and the system queries the openshift server to vet

## Initial questions to address:

- What are the recommended specs for an SNO VM? 
- What openshift certifications are there, and is there an recommended order? 

## Initial tasks

- Initialize as a git repository 
- Create a new public github repo using the `gh` CLI and targetted ssh as the remote endpoint type
- Set up the Bob Software Factory to be how work is tracked for this project. 
- Create initial tasks to start building our learning lab 
