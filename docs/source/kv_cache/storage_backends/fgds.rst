Fgds Backend
==================

.. _Fgds-overview:

Overview
--------

This backend will work with Fgds-based filesystem. The `Fgds <https://github.com/Storage-and-OS-for-AI/fgds>`_ is a refactored
I/O Stack for GPU Direct Storage without Phony Buffers, which has been accepted for SC and the performance of I/O latency
and bandwidth is better than gds.

Ways to configure LMCache Fgds Backend
-----------------------------------------

**1. Environment Variables:**

.. code-block:: bash

    # 256 Tokens per KV Chunk
    export LMCACHE_CHUNK_SIZE=256
    # Path to store files
    export LMCACHE_FGDS_PATH="/mnt/fgds/cache"
    # Fgds Buffer Size in MiB
    export LMCACHE_FGDS_BUFFER_SIZE="8192"
    # Disabling CPU RAM offload is sometimes recommended as the
    # CPU can get in the way of GPUDirect operations
    export LMCACHE_LOCAL_CPU=False

**2. Configuration File**:

Passed in through ``LMCACHE_CONFIG_FILE=your-lmcache-config.yaml``

Example ``config.yaml``:

.. code-block:: yaml

    # 256 Tokens per KV Chunk
    chunk_size: 256
    # Disable local CPU
    local_cpu: false
    # Path to file system of Fgds-enabled mount
    fgds_path: "/mnt/fgds/cache"
    # Fgds Buffer Size in MiB
    fgds_buffer_size: 8192


Fgds Buffer Size Explanation
------------------------------

The backend currently pre-registers buffer space to speed up fgds operations. This buffer space
is registered in VRAM so options like ``--gpu-memory-utilization`` from ``vllm`` should be considered
when setting it. For example, a good rule of thumb for H100 which generally has 80GiBs of VRAM would
be to start with 8GiB and set ``--gpu-memory-utilization 0.85`` and depending on your workflow fine-tune
it from there.


Setup Example
-------------

.. _fgds-prerequisites:

**Prerequisites:**

- A Machine with at least one GPU. You can adjust the max model length of your vllm instance depending on your GPU memory.

- A mounted file system. A file system supporting Fgds will work best.

- Deploy the Python API module of fgds following the `instructions <https://github.com/Storage-and-OS-for-AI/fgds/blob/v1.0/python/README.md>`_.

- vllm and lmcache installed (:doc:`Installation Guide <../../getting_started/installation>`)

- Hugging Face access to ``meta-llama/Llama-3.1-8B-Instruct``

.. code-block:: bash

    export HF_TOKEN=your_hugging_face_token

**Step 1. Check fgds filesystem:**

To check if the fgds is ready, use `example` from fgds project:

.. code-block:: bash

    sudo /path/to/fgds/build/bin/example

Create a directory under the file system mount (the name here is arbitrary):

.. code-block:: bash

    mkdir /mnt/fgds/cache

**Step 2. Start a vLLM server with file backend enabled:**

Create an lmcache configuration file called: ``fgds-backend.yaml``

.. code-block:: yaml

    local_cpu: false
    chunk_size: 256
    fgds_path: "/mnt/fgds/cache"
    fgds_buffer_size: 8192

If you don't want to use a config file, uncomment the first three environment variables
and then comment out the ``LMCACHE_CONFIG_FILE`` below:

.. code-block:: bash

    # LMCACHE_LOCAL_CPU=False \
    # LMCACHE_CHUNK_SIZE=256 \
    # LMCACHE_FGDS_PATH="/mnt/fgds/cache" \
    # LMCACHE_FGDS_BUFFER_SIZE=8192 \
    LMCACHE_CONFIG_FILE="fgds-backend.yaml" \
    vllm serve \
        meta-llama/Llama-3.1-8B-Instruct \
        --max-model-len 65536 \
        --kv-transfer-config \
        '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'