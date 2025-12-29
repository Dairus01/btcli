# Faucet and PoW Registration Configuration

## Problem
The `btcli wallet faucet` and `btcli subnets pow_register` commands require numerous arguments to configure the Proof of Work (PoW) process effectively. These arguments include:
- `processors`
- `update_interval`
- `output_in_place`
- `verbose`
- `use_cuda`
- `dev_id`
- `threads_per_block`
- `max_successes` (for faucet)

Users frequently running these commands (e.g., on local testnets) found it cumbersome to repeatedly specify these flags every time they ran a command. There was no mechanism to persist these preferences in the global configuration file (`config.yml`).

## Thought Process
To improve the user experience, we needed to integrate these parameters into the existing configuration system. The goal was to establish a hierarchy of precedence for parameter resolution:
1.  **CLI Arguments**: Explicit flags provided at runtime (e.g., `--processors 4`) should always take precedence.
2.  **Configuration File**: If no CLI flag is provided, the system should check the `~/.bittensor/config.yml` file.
3.  **Defaults**: If neither are present, fall back to the hardcoded system defaults.

### Key Considerations:
-   **Config Schema**: The `Defaults` class in `bittensor_cli/src/__init__.py` defines the default structure of the configuration. New keys needed to be added here.
-   **Typer `Options`**: The `Options` class in `bittensor_cli/cli.py` defines reusable CLI arguments. Currently, these were defined locally within the command functions or using default values that obscured whether the user *actually* passed a flag or if it was just the default. To support the precedence logic, the `Options` needed to default to `None` so the code could distinguish between "user input" and "no input".
-   **Config Management**: The `config set` and `config clear` commands needed updates to expose these new options to the user.

## Solution
The implementation involved the following steps:

1.  **Update Defaults**:
    Added the following keys to `Defaults.config.dictionary` in `bittensor_cli/src/__init__.py`:
    -   `pow_register_processors`
    -   `pow_register_update_interval`
    -   `pow_register_output_in_place`
    -   `pow_register_verbose`
    -   `pow_register_use_cuda`
    -   `pow_register_dev_id`
    -   `pow_register_threads_per_block`
    -   `faucet_max_successes`

2.  **Define Reusable Options**:
    Added `typer.Option` definitions for these parameters in the `Options` class within `bittensor_cli/cli.py`. Crucially, these are initialized to `None`.

3.  **Update Command Logic**:
    Modified `wallet_faucet` and `subnets_pow_register` in `bittensor_cli/cli.py`. The logic now checks if the argument is `None`. If so, it attempts to retrieve the value from `self.config`. If that is also `None`, it falls back to the original default values from `bittensor_cli.src.defaults`.

    ```python
    if processors is None:
        processors = (
            self.config.get("pow_register_processors")
            if self.config.get("pow_register_processors") is not None
            else defaults.pow_register.num_processes
        )
    ```

4.  **Update Config Tools**:
    Updated `set_config` and `del_config` in `CLIManager` to accept these new parameters, allowing users to easily manage them via the CLI:
    -   `btcli config set --processors 4 --use-cuda`
    -   `btcli config clear --processors`

This solution provides a seamless experience where users can configure their environment once and run commands succinctly, while retaining the flexibility to override settings on the fly.
