ec2-docker-notebook
====================

.. contents::

Summary
--------
Running Jupyter Notebook on Docker on Amazon EC2 using Docker Machine

Steps
------

Install Docker
~~~~~~~~~~~~~~~
See https://docs.docker.com/engine/installation/

Install Docker Machine
~~~~~~~~~~~~~~~~~~~~~~~
See https://docs.docker.com/machine/install-machine/

Obtain AWS access key
~~~~~~~~~~~~~~~~~~~~~~
Assuming you already signed up for AWS.

1. Follow `AWS documentation <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/get-set-up-for-amazon-ec2.html#create-an-iam-user>`_ to create a user in an administrator group, and obtain the access key.

  Note: Leave **Generate an access key for each user** box checked while creating user to obtain the access key. **Assign a custom password** (step 8-9) is not necessary.

2. Create file ``~/.aws/credentials`` with the access key:

   .. code-block::

    [default]
    aws_access_key_id = AKID1234567890
    aws_secret_access_key = MY-SECRET-KEY

 This will be used by Docker Machine automatically.

Create an EC2 instance
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

 docker-machine create --driver amazonec2 aws-box

- This will create an ``t2.micro`` instance of image ``ami-26d5af4c`` (Ubuntu 15.10) in region ``us-east-1`` (N. Virginia) with 16GB EBS volume.
- Instance will be attached to a security group, named ``docker-machine``, with inbound rules for SSH (port 22) and Docker (port 2376).
- Docker will be installed on the instance
- See https://docs.docker.com/machine/drivers/aws/#/options for flags to customize the instance.

Output may look like this:

.. code-block::

 Running pre-create checks...
 Creating machine...
 (aws-box) Launching instance...
 Waiting for machine to be running, this may take a few minutes...
 Detecting operating system of created instance...
 Waiting for SSH to be available...
 Detecting the provisioner...
 Provisioning with ubuntu(systemd)...
 Installing Docker...
 Copying certs to the local machine directory...
 Copying certs to the remote machine...
 Setting Docker configuration on the remote daemon...
 Checking connection to Docker...
 Docker is up and running!
 To see how to connect your Docker Client to the Docker Engine running on this virtual machine, 
 run: docker-machine env aws-box

Enable HTTPS port on the EC2 instance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default, the AWS driver in Docker Machine attaches the EC2 instance to security group named ``docker-machine`` (automatically created if not existed) with only SSH, and Docker ports. We want to add HTTPS port (443) for Jupyter Notebook.

To do so, navigate to `Security Groups in EC2 console <https://console.aws.amazon.com/ec2/v2/home?#SecurityGroups:sort=groupId>`_ and add an inbound rule for HTTPS.

================= ========== ============ ============
 Type              Protocol   Port Range   Source
----------------- ---------- ------------ ------------
 SSH               TCP        22           0.0.0.0/0
 Custom TCP Rule   TCP        2376         0.0.0.0/0
 HTTPS             TCP        443          0.0.0.0/0
================= ========== ============ ============

Activate the EC2 host
~~~~~~~~~~~~~~~~~~~~~~

We need to tell our (local) Docker client that we want it to communicate with the Docker server on the ``aws-box``. So we need to make it `active`:

.. code-block::

 eval $(docker-machine env aws-box)

- All it does is setting 4 environment variables: ``DOCKER_TLS_VERIFY``, ``DOCKER_HOST``, ``DOCKER_CERT_PATH``, ``DOCKER_MACHINE_NAME``.
- You can dry run ``docker-machine env aws-box`` to see what are set
- Verify with ``docker-machine ls``

  .. code-block::

   NAME      ACTIVE   DRIVER      STATE     URL                       SWARM   DOCKER    ERRORS
   aws-box   *        amazonec2   Running   tcp://x.x.x.x:2376           v1.12.1

Run scipy-notebook
~~~~~~~~~~~~~~~~~~~

`scipy-notebook <https://github.com/jupyter/docker-stacks/tree/master/scipy-notebook>`_ is an image created by Jupyter for scientific computations.

Here we remotely deploy the Notebook stack onto the EC2 instance:

.. code-block::

 docker run -d -p 443:8888 --name notebook \
            -e USE_HTTPS=yes jupyter/scipy-notebook \
            start-notebook.sh \
            --NotebookApp.password='sha1:xxxxxxxxxxx:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'

- Our (local) Docker client will ask the Docker server on ``aws-box`` to fetch the image (``jupyter/scipy-notebook``), and start it
- ``-p 443:8888`` publishes Docker container's port 8888 as port 443 on ``aws-box``
- ``-e USE_HTTPS=yes`` makes Jupyter Notebook use SSL. By default, self-signed certificates are created.
- ``--NotebookApp.password='sha1:...'`` sets the password for the notebook. See `Jupyter documentation <https://jupyter-notebook.readthedocs.io/en/latest/public_server.html#preparing-a-hashed-password>`_ on how to create this password hash.

That is it!

You can now access the notebook via https://x.x.x.x (whatever your EC2 instance IP is)

You can find out the IP using ``docker-machine ip aws-box`` or from the AWS console.
