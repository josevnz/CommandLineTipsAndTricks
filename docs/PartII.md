# A few command line tricks you can learn faster than drinking your early coffee

![](mazinger-z.png)

In this short tutorial I want to share with you a few tricks and tips to deal with some common situations. We will cover this:

* Reading files with Bash

I will present you with a challenge and the tools demonstrating how to solve each problem.

## What you will need for this tutorial?

* A Linux distribution
* Curiosity

## Reading a file, line by line, with Bash

Say than you have a file with data and you want to read it line by line, to store it in say a dictionary:

```shell
josevnz@orangepi5:cat /etc/os-release
NAME="Fedora Linux"
VERSION="37 (Workstation Edition)"
ID=fedora
VERSION_ID=37
VERSION_CODENAME=""
PLATFORM_ID="platform:f37"
PRETTY_NAME="Fedora Linux 37 (Workstation Edition)"
ANSI_COLOR="0;38;2;60;110;180"
LOGO=fedora-logo-icon
CPE_NAME="cpe:/o:fedoraproject:fedora:37"
DEFAULT_HOSTNAME="fedora"
HOME_URL="https://fedoraproject.org/"
DOCUMENTATION_URL="https://docs.fedoraproject.org/en-US/fedora/f37/system-administrators-guide/"
SUPPORT_URL="https://ask.fedoraproject.org/"
BUG_REPORT_URL="https://bugzilla.redhat.com/"
REDHAT_BUGZILLA_PRODUCT="Fedora"
REDHAT_BUGZILLA_PRODUCT_VERSION=37
REDHAT_SUPPORT_PRODUCT="Fedora"
REDHAT_SUPPORT_PRODUCT_VERSION=37
SUPPORT_END=2023-12-05
VARIANT="Workstation Edition"
VARIANT_ID=workstation
```

Wouldn't be nice if we could say ``` echo ${my_dict["PRETTY_NAME"]}``` to get the value of "Fedora Linux 37 (Workstation Edition)", without the quotes?


### Using Shell built-in

You can acomplish this task with pure Bash:

```shell
declare -A _GLOBAL_MAP  # Declare a dictionary, scope is global to whoever calls this script.
export _GLOBAL_MAP
function populate_global_map() {
    local data_file
    local separator
    data_file="${1}"
    separator="${2:-"="}"  # Provide a default token separator
    if [[ -f "$data_file" ]]; then
        while IFS="$separator" read -r key val; do  # Read each line into two variables
            val=${val/\"/}  # Remove opening '"'
            val=${val/\"/}  # Remove closing '"'
            _GLOBAL_MAP["$key"]="$val"  # Assign new value to key on dictionary
        done < "$data_file"  # Feed the file contents to our read command above, on a loop
    fi
}
```

Not bad, just 14 lines of code with some basic validations. Say you save this on a file called ~/dict_from_file.sh

```shell
# Source the function
. ~/dict_from_file
# Let's read the contents of /etc/os-release
populate_global_map /etc/os-release
# Show all the keys on '_GLOBAL_MAP'
echo ${!_GLOBAL_MAP[@]}
PLATFORM_ID DEFAULT_HOSTNAME LOGO NAME REDHAT_BUGZILLA_PRODUCT REDHAT_BUGZILLA_PRODUCT_VERSION BUG_REPORT_URL HOME_URL REDHAT_SUPPORT_PRODUCT SUPPORT_END CPE_NAME VERSION_CODENAME ANSI_COLOR ID VARIANT PRETTY_NAME DOCUMENTATION_URL VARIANT_ID SUPPORT_URL VERSION_ID VERSION REDHAT_SUPPORT_PRODUCT_VERSION
# Show all the values on '_GLOBAL_MAP'
echo ${_GLOBAL_MAP[@]}
platform:f37 fedora fedora-logo-icon Fedora Linux Fedora 37 https://bugzilla.redhat.com/ https://fedoraproject.org/ Fedora 2023-12-05 cpe:/o:fedoraproject:fedora:37 0;38;2;60;110;180 fedora Workstation Edition Fedora Linux 37 (Workstation Edition) https://docs.fedoraproject.org/en-US/fedora/f37/system-administrators-guide/ workstation https://ask.fedoraproject.org/ 37 37 (Workstation Edition) 37
# Show a specific key value
echo "PRETTY_NAME = ${_GLOBAL_MAP["PRETTY_NAME"]}"
PRETTY_NAME = Fedora Linux 37 (Workstation Edition)
```

## What is next

There is _so much more to explore_. The tips above introduced you to some important concepts, so why not to learn much more about them?

* RTF  [(Read the Fine Manual)](https://www.gnu.org/software/bash/manual/html_node/Arrays.html)
* Source code for this tutorial can be found [here](https://github.com/josevnz/CommandLineTipsAndTricks).
