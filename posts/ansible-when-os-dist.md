# ansible tasks based on operating system (os)

tags: devops, ansible, linux, ubuntu, python, pip

date: mon sep 28 2020

## Intro
Got tired of running ansible against different versions of Ubuntu and having to modify my tasks on the fly to deal with python's shit. In 20, pip2 is not an apt package, but in 18, it is. Figured Ansible in all its glory should have a way to deal with this programmatically, so I set off on a quick 10 mins (read: 3 hours) endeavour to get that fixed. 

## The answer

In the end, this does the trick:

```
- name: install python-pip
  apt:
      name: python-pip
      state: present
  when: "ansible_lsb.major_release == '18'"
```

Note, this is with Ansible version 2.5 (yes, matters)

To get there however, besides major syntax errors, such as trying `"{{ansible_distribution_major_release}} == ..."`, I kept getting the error that the `'ansible_distribution_major_version' is undefined`. To fix this, I tried adding in `gather_facts: True/true/yes/Yes` to the playbook calling the task. I believe this is being done by default in my current Ansible version, however it wasn't accepting the dict. 

There is a way to lookup the facts though, and print them as debug messages.

Before we get into that though, lets take a moment to ponder wtf there is both `ansible` and `ansible-playbook`, or even more, why they have different flags, often for the same purpose

...

Okay, now that we've level upped our rage, here are some wins:

- Print all facts

```
ansible x.x.x.x -u root -i hosts --private-key /home/user/.ssh/id_rsa -m setup -e 'ansible_python_interpreter=/usr/bin/python3'
```

- Debug print within the playbook/task

```
- debug:
    msg: "ansible_lsb.major_release: {{ansible_lsb.major_release}}"
```

So the answer was that I had the wrong dict name (got the wrong one from about 20 different posts/stackoverflows). The wrong is `ansible_distribution_major_release` and the correct one is `ansible_lsb.major_release`. Unless you're lucky, this is going to differ for you. 

Hope to save someone some time ...

## update

Dec 18, 2020

Using the following instead:

```
when: ansible_lsb.id != "Kali"
```

And when using a more recent version of Ansible (current: 2.5) such as v2.9, this works:

```
when: ansible_distribution == "Kali GNU/Linux"
```