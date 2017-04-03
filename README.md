# UK Polling Stations build and deploy

## Amazon Machine Images (AMI)

We user [packer] to build golden AMIS. On OSX this can be installed via `brew
install packer`. The AMIs we build are designed to come up as quickly as
possible so they can serve traffic. This means that they contain the entier
AddressBase db pre-imported. To make code-only changes quicker we have two
AMIS:

- The "imported DB" AMI which has postgres and the DB imported but nothing
  else.
- The code/app AMI which is built off the DB ami. This is much faster to build
  as we don't have to wait for the multi gigabyte Postgres dump to restore.


### How to build

- If the addressbase dump has changed, or if you haven't built it yet then:

#### 1. Create a DB dump containing addressbase

You will need a local working copy of the app code and a fresh copy of addressbase (basic) in CSV files. clean and import address base according to the app's documentation, at the moment this involves running `./manage.py clean_addressbase` and `./manage.py import_cleaned_addresses`.

#### 2. Export and upload addressbase

    pg_dump --column-inserts --no-privileges --no-tablespaces --file "/tmp/addresses.sql.tar" --table "pollingstations_pollingdistrict" --table "addressbase_address" -F c "polling_stations"

Upload that file to s3://pollingstations-packer-assets/addressbase/addressbase.sql.tar

#### 3. Make the image
        AWS_PROFILE=democlub ./packer addressbase

  That will output an AMI id that you will need to manually copy. Look for
  lines like this near the end of the output:

      ==> database: Creating the AMI: addressbase 2016-06-20T12-38-27Z
          database: AMI: ami-7c8f160f

  (This might take a *long* time at the "Waiting for AMI to become ready..."
  stage)

  The AMI id (`ami-7c8f160f` in this case) needs to go into
  `./packer-vars.json` under the database_ami_id.

- If the pollingstations/council dump has changed then similary to
    `addressbase` above run

        AWS_PROFILE=democlub ./packer imported-db

  and store the resulting AMI id in the `imported-db_ami_id` key.

  This will build one the addressbase AMI and include in the councils and
  polling stations data.

- To make just a code change the build the `server` AMI. This is built on the
    `imported-db` AMI and just adds the code changes.

        ./packer server

  **NOTE**: To run this you will need the Ansible Vault password. Place it in
  a `.vault_pass.txt` file in the same directory as this file.

### Debugging the build

If something is going wrong with the ansible or other commands you run on the
build then the best tool for debugging it is to SSH onto the instance. By
default packer will terminate the instance as soon as it errors though. If you
add `-debug` flag then packer will pause at each stage until you hit enter,
giving you the time to SSH in and examine the server.

Packer will generate a per-build SSH private key which you will have to use:

    ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no  \
      -l ubuntu -i ./ec2_server.pem 54.229.202.43

(The IP address will change every time too - get that from the output.)


[packer]: https://www.packer.io/

## Ad-hoc tasks

Sometimes you need to fix something urgently and you don't want to wait for an
AMI rebuild. In which case you can use the ansible dynamic inventory:

    ANSIBLE_HOST_KEY_CHECKING=False AWS_PROFILE=democlub \
    ansible -i dynamic-inventory/ \
      -b --become-user polling_stations \
      tag_Env_prod \
      -a id

The above command will run `id` as the polling_stations user. This invokes the
[command module][ansible_command_module] - you might want to add `-a shell` if
your command is more complex.

What ever change you make don't forget to roll it back into the AMI, create a
new launch config and update the ASG config (even if you don't replace the
existing instances) otherwise a scaling event or failed host will "revert"
your changes.

[ansible_command_module]: http://docs.ansible.com/ansible/command_module.html
