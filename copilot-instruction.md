I have to create a workshop with a cutom sdk that includes:
- The skills i created to pack new inference snaps. These skill you can copy them from ~/.agents/skills/ and they are 4 inference-snap-build-and-prompt-check, inference-snap-from-example inference-snap-static-checks and finally inference-snap-structure
- The sdk should also include an installation of opencode, performed like official opencode documentatino asks, `curl -fsSL https://opencode.ai/install | bash`

Please use the sdk-designer skill.
Avoid modofying the file workshop, just create the sdk and give instruction on how to use it in the workshop.yaml file.
Keep the sdk readable and minimalel, only include the necessary steps to install the skills and opencode.