# LeRobot AI Set Up

## Set Up Environment (ensure you run on Linux):

Choose a root folder and in the terminal run the following commands in order:

```bash
wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"

bash Miniforge3-$(uname)-$(uname -m).sh

conda create -y -n lerobot python=3.12

conda activate lerobot

conda install ffmpeg -c conda-forge

git clone https://github.com/huggingface/lerobot.git

cd lerobot

pip install -e ".[core_scripts]"

pip install -e ".[training]"

pip install -e ".[all]"

pip install lerobot

pip install -e ".[feetech]"
```

## Construct Arm:

Follow detailed tutorial here: [https://huggingface.co/docs/lerobot/main/en/so101](https://huggingface.co/docs/lerobot/main/en/so101)

*(This construction tutorial is SO-101 arm specific)*

---

## Test Arms with Teleoperation (optional):

```bash
lerobot-teleoperate \
    --robot.type=so101_follower \
    --robot.port=/dev/tty.usbmodem58760431541 \
    --robot.id=my_awesome_follower_arm \
    --teleop.type=so101_leader \
    --teleop.port=/dev/tty.usbmodem58760431551 \
    --teleop.id=my_awesome_leader_arm
```

**Note:** Text like this is meant to be modified to fit your personal environment.

---

## Create HuggingFace Account and Link:

To obtain token, make account and click on access tokens under account drop down in top right corner of website.

```bash
hf auth login --token ${HUGGINGFACE_TOKEN} --add-to-git-credential
```

> **INFO:** Huggingface-hub should be installed via the environment set-up, but if not you may need to install it manually to run this command

---

## Record Dataset:

```bash
lerobot-record \
    --robot.type=so101_follower \
    --robot.port=/dev/tty.usbmodem4210431551 \
    --robot.id=my_awesome_follower_arm \
    --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}}" \
    --teleop.type=so101_leader \
    --teleop.port=/dev/tty.usbmodem58760431551 \
    --teleop.id=my_awesome_leader_arm \
    --dataset.repo_id=${HF_USER}/record-test \
    --dataset.num_episodes=5 \
    --dataset.single_task="Grab the black cube"
```

Where each variable in the command above needs to be adapted to your environment. Find the robot ports by using the command `lerobot-find-port` and the `robot.id` / `teleop.id` should be the id you customized during configuration (it most likely points to the YAML configuration file). 

The `repo.id` should correspond to your huggingface username and a repo you wish to create. For example, my username is Basr88, so my `repo.id` may be `Basr88/My_new_dataset`.

> **IMPORTANT:** LeRobot’s ACT model takes a minimum of 640x480 camera input. Anything less and the model will not function properly. This is important information for choosing which cameras to utilize. In addition, the specified camera fps directly determines robot movement speed as the model runs at a default 30 fps, meaning if a 15 fps is specified in the camera, it will cause the robot’s movements to be twice as fast as the training input.

---

## Training Tips

When choosing tasks and camera angles for the AI to learn, the methodology for choosing a valid task is deciding whether or not it can be performed by a human using **ONLY** the provided camera input. If not, an additional camera angle or change in positioning may be needed for a successful model. 

Two cameras are almost always better than one, as they provide more depth knowledge and situational awareness, however this comes at the cost of increased training time. 

It can be beneficial to split training sessions into 25 episode increments. Anything more and the recording program begins to slow down substantially. 

Datasets can be merged using the CLI command:
```bash
lerobot-edit-dataset \
    --repo_id lerobot/pusht_merged \
    --operation.type merge \
    --operation.repo_ids "['lerobot/pusht_train', 'lerobot/pusht_val']"
```

When trying to build a resilient model, it is important to introduce variability to anything that is not important during training. This includes altering the background, adding new items to a table, etc. 

For example, if there is a sticky note on the wall, it may be that the model assigns a high weight to that sticky note because it never realizes the sticky note isn’t important. Once trained, if the sticky note is removed, however, the model would fail in this scenario. 

---

## Accessing Dataset:

To access the recorded dataset, LeRobot automatically saves it into the computers `/.cache` folder under huggingface and the `repo.id` specified. You can either move this file for training manually or upload it to the HuggingFace Hub. 

This process is automatic if trained while connected to WiFi, however can be done manually using:

```bash
hf upload ${HF_USER}/record-test ~/.cache/huggingface/lerobot/{repo-id} --repo-type dataset
```

---

## Training Model from Dataset:

```bash
lerobot-train \
  --dataset.repo_id=${HF_USER}/so101_test \
  --policy.type=act \
  --output_dir=outputs/train/act_so101_test \
  --job_name=act_so101_test \
  --policy.device=cuda \
  --wandb.enable=false \
  --policy.repo_id=${HF_USER}/my_policy
```

### Editable fields are:

* **Dataset.repo_id**: The repo.id of your trained dataset (can be local file directory or repo.id from Huggingface Hub)
* **Output_dir**: The directory for final model output to be stored
* **Job_name (Optional)**: Name for job in terminal
* **Wandb.enable (Optional)**: If true, must link to personal wandb account to get real-time training weights and updates 
* **Policy.repo_id**: The desired repo.id for the final trained model.

> **INFO:** While the default training steps are 100000, you can change this by adding the line `--steps=120000`. The more steps given, the further the model adapts to the dataset data. Too little and the AI cannot perform inference, too much and the AI cannot adapt to new scenarios. 
> 
> Sticking to 100000 steps seems to be optimal in our testing. 

---

## Running the Model:

```bash
lerobot-rollout \
  --strategy.type=base \
  --policy.path=${HF_USER}/my_policy \
  --robot.type=so100_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.cameras="{ front: {type: opencv, index_or_path: /dev/video10, width: 640, height: 480, fps: 30}}" \
  --task="Put lego brick into the transparent box" \
  --duration=60
```

Match these values to the values of the dataset and provide the `policy.path` as the trained model’s `repo.id`. This should run inference and you can evaluate the trained model.

---

## IMPORTANT SIDE NOTE

While making this documentation, the LeRobot library and online documentation has changed. It is important to reference the online documentation alongside this one to ensure up-to-date information. This also means that most online forums and AI will provide out-of-date information, which is something we ran into a lot. Once you have a working LeRobot environment, try to keep it self-contained and rely on external computers as little as possible. Versioning conflicts with LeRobot are a mess.
