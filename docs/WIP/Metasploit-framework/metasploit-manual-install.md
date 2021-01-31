# Metasploit-Framework installation for development

Metasploit-Framework installation for development - WIP

-----

## Date

| Action | Date |
| -- | -- |
|Post date|2020/09/27|
|Last update|2020/09/27|

-----

## Installation

!!! note "`install_metasploit.sh`"
    ```bash

    # NOT FINISHED YET

    [ $UID -ne 0 ] && printf 'run this as root or with sudo\n' && exit 1

    # SET TO 1 IF YOU USE RBENV
    use_rbenv=0
    RBENV_ROOT=/opt/rbenv

    # Change these values if they are not the same
    _uid=1000
    _gid=1000

    _usrname=$(id -un 1000)
    _grpname=$(id -gn 1000)

    #########################

    apt-get update && apt-get upgrade --fix-missing && apt-get dist-upgrade --fix-missing # Check, reboot if kernel upgrade, etc.

    apt-get install -y bundler git libpq-dev libpcap-dev autoconf build-essential zlib1g-dev libsqlite3-dev postgresql postgresql-client

    [ "$(which setfacl)" == "" ] && apt-get install -y acl

    if  [ ! -d /DATA ]; then
        mkdir /DATA
        chown "$_uid":"$_gid" /DATA
        chmod 750 /DATA
    fi

    # Then restore backups in /DATA

    # Make an msf4 shared directory in which root and your user can collaborate without any issue
    # Can be usefull when working on msf with both root and a trusted user
    if  [ ! -d /DATA/msf4 ]; then
        mkdir /DATA/msf4
        chown "$_uid":"$_gid" /DATA/msf4
        chmod 2770 /DATA/msf4
        setfacl -d -m g::rwx /DATA/msf4
        setfacl -d -m o::--- /DATA/msf4
        
        [ -d /root/.msf4 ] && mv /root/.msf4 /root/.msf4.ORIG
        [ -d /home/"$_usrname"/.msf4 ] && mv /home/"$_usrname"/.msf4 /home/"$_usrname"/.msf4.ORIG
        
        ln -s /DATA/msf4 /root/.msf4
        ln -s /DATA/msf4 /home/"$_usrname"/.msf4
    fi

    git clone https://github.com/rapid7/metasploit-framework.git /opt/metasploit-framework

    cat > /opt/metasploit-framework/Gemfile.local <<'EOF'
    # Include the Gemfile included with the framework. This is very
    # important for picking up new gem dependencies.
    msf_gemfile = File.join(File.dirname(__FILE__), 'Gemfile')
    if File.readable?(msf_gemfile)
    instance_eval(File.read(msf_gemfile))
    end

    # Create a custom group
    group :local do
    gem 'wirble'
    gem 'awesome_print'
    gem 'code'
    gem 'core_docs'
    end
    EOF

    ##########################

    # Create msf legacy scripts to redirect old tools
    cat > /opt/metasploit-framework/script-exploit <<'EOF'
    #!/bin/sh

    name_script=$(basename $0)
    tool_name=${name_script##msf-}

    exec /usr/share/metasploit-framework/tools/exploit/$tool_name.rb "$@"
    EOF

    ##########################

    cat > /opt/metasploit-framework/script-password <<'EOF'
    #!/bin/sh

    name_script=$(basename $0)
    tool_name=${name_script##msf-}

    exec /usr/share/metasploit-framework/tools/password/$tool_name.rb "$@"
    EOF

    ##########################

    cat > /opt/metasploit-framework/script-recon <<'EOF'
    #!/bin/sh

    name_script=$(basename $0)
    tool_name=${name_script##msf-}

    exec /usr/share/metasploit-framework/tools/recon/$tool_name.rb "$@"
    EOF

    ##########################

    chmod 755 /opt/metasploit-framework/script-exploit
    chmod 755 /opt/metasploit-framework/script-password
    chmod 755 /opt/metasploit-framework/script-recon

    cd /opt/metasploit-framework

    #####################################

    if [ $use_rbenv -eq 1 ]; then
        # Set up rbenv for $_usrname and root
        apt-get install rbenv

        mkdir "$RBENV_ROOT"
        chown "$_uid":"$_gid" "$RBENV_ROOT"
        chmod 2775 "$RBENV_ROOT"
        setfacl -d -m g::rwx "$RBENV_ROOT"
        setfacl -d -m o::r-x "$RBENV_ROOT"

        echo "export RBENV_ROOT=$RBENV_ROOT" | tee -a /root/.bashrc /home/"$_usrname"/.bashrc >/dev/null
        echo 'eval "$(rbenv init -)"'        | tee -a /root/.bashrc /home/"$_usrname"/.bashrc >/dev/null

        export RBENV_ROOT="$RBENV_ROOT"
        eval "$(rbenv init -)"

        # Installing a new ruby version can take a long time
        rbenv install
    fi

    bundler_version=$(grep 'BUNDLED WITH' -A1 /opt/metasploit-framework/Gemfile.lock | sed '1d' | tr -d ' ')
    [ "$bundler_version" == "" ] && printf 'Error: could not determine bundler version\n' >&2 && exit 1

    gem install bundler:"$bundler_version"
    [ $? -ne 0 ] && printf 'Error: could not install bundler version: %s\n' "$bundler_version" >&2 && exit 1

    # Install gems system-wide. Bundler is good at dependency management
    # bundle config --local path vendor/bundle
    bundle config --local without "development test"

    if [ $use_rbenv -eq 1 ]; then
        # This will install gems in $RBENV_ROOT
        rbenv exec bundle install --gemfile=Gemfile.local --jobs=4
        
        # Mimic that it was your user that ran the install
        # Thanks to setfacl and SetGID bit on $RBENV_ROOT, it will not break anything
        chown "$_uid":"$_gid" -R "$RBENV_ROOT"
    else
        bundle install --gemfile=Gemfile.local --jobs=4
    fi

    chown "$_uid":"$_gid" -R /opt/metasploit-framework

    msf_alternatives=$(update-alternatives --get-selections | grep 'metasploit' | awk '{print $1}')

    for i in ${msf_alternatives[@]}; do
        update-alternatives --remove-all "$i"
        update-alternatives --install /usr/bin/"$i" "$i" /opt/metasploit-framework/"$i" 60
    done

    OLD_IFS=$IFS
    IFS=$'\n'
    for i in $(ls -l /usr/bin/msf-*); do
        link_name=$(echo "$i" | awk '{print $9}')
        target_name=$(echo "$i" | awk '{print $11}')
        
        new_target=/opt/metasploit-framework/$(basename "$target_name")
        rm -f "$link_name"
        ln -s "$new_target" "$link_name"
    done
    IFS=$OLD_IFS

    # Setup msf database
    systemctl restart postgresql
    systemctl enable postgresql

    # TODO:
    # Add pg_ctl to PATH
    # Add initdb to PATH
    # Run: cd /opt/metasploit-framework && ./msfdb init

    ```

-----

## IRB setup for Metasploit-Framework

Once installed we need to allow Metasploit Framework to load them since they are not in the Gemset of the project and we can not add them to the Gemset file since msfupdate will replace the file. To allow this in the project folder we need to create a Gemfile.local file loading the current project gems and our additional gems.

With this setup:

- better console printing with `wirble` module
- abuse the `awesome_print` module for better output, like `ap self.sys.methods`
- parse code with the `code` module, like `Code.for framework.db, :report_cred`
- use the function `local_methods` instead of `methods` to show local methods only, excluding instance methods

??? note "`/opt/metasploit-framework/Gemfile.local`"
    ```ruby
    # Include the Gemfile included with the framework. This is very
    # important for picking up new gem dependencies.
    msf_gemfile = File.join(File.dirname(__FILE__), 'Gemfile')
    if File.readable?(msf_gemfile)
    instance_eval(File.read(msf_gemfile))
    end

    # Create a custom group
    group :local do
    gem 'wirble'
    gem 'awesome_print'
    gem 'code'
    gem 'core_docs'
    end
    ```

??? note "`~/.irbrc`"
    ```ruby
    # Print message to show that irbrc loaded.
    puts '~/.irbrc has been loaded.'

    # Load wirb to colorize the console
    require 'wirble'
    Wirble.init
    Wirble.colorize

    # Load awesome_print.
    require 'ap'

    # Load the Code gem 
    require 'code'


    # Remove the annoying irb(main):001:0 and replace with >>
    IRB.conf[:PROMPT_MODE]  = :SIMPLE

    # Tab Completion
    require 'irb/completion'

    # Automatic Indentation
    IRB.conf[:AUTO_INDENT]=true

    # Save History between irb sessions
    require 'irb/ext/save-history'
    IRB.conf[:SAVE_HISTORY] = 100
    IRB.conf[:HISTORY_FILE] = "#{ENV['HOME']}/.irb-save-history"

    # get all the methods for an object that aren't basic methods from Object
    class Object
    def local_methods
        return (methods - Object.instance_methods)
    end
    end
    ```

After those files were created, install the Gemfile.local in the metasploit project (as stated [here](https://gist.github.com/kn0/11129288), it needs to be installed manually, and probably again after each update)

**Could not install this Gemfile.local without fcking up metasploit-framework installation in last Kali (msf5)**

-----

## Sources

[Metasploit IRB setup](https://www.darkoperator.com/blog/2017/10/21/basics-of-the-metasploit-framework-irb-setup)

-----
