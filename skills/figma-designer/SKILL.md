---
name: figma-designer
description: Interact with Figma to assist with design tasks.
allowed-tools: Read, Grep, Glob, navigate_page, evaluate_script
---

# Figma Designer Skill

This skill allows you to interact with Figma to assist with various design tasks.
 
## Instructions 
You'll be interacting with Figma via the web browser. First, you'll want to
1. Use the `navigate_page` tool to go to [Figma's website](https://www.figma.com/). Then, prompt the user to log in to their Figma account if they are not already logged in and open a specific design file.
2. Once the user has opened the design file, you can use the `evaluate_script` to confirm that you have access to the figma global object, which indicates that you are within the Figma environment. If you do not have access, inform the user that they need to open a Figma design file or see <troubleshooting>
3. After confirming access, execute the user's query using the `evaluate_script` tool to run JavaScript code that interacts with the Figma plugin API. You can perform tasks such as creating shapes, modifying properties, or extracting information from the design file. 

## Rules of Engagement
- Always explain in plain english what you are about to do. Assume that the user cannot read code.
- Do not try to alternative solutions like using the REST API or manually interacting with the Figma UI

<troubleshooting>
If `figma is not defined`, make sure that the user has appropriate permissions to edit the file and run plugins. If the user doesn't, suggest creating a new branch on the file. If the `figma` global is still not accessible, instruct the user to manually open any plugin and close it, then try again. There is a weird bug where the `figma` global is not available until a plugin has been opened at least once in the file.
</troubleshooting>

## Additional Documentation

The full reference to the Figma plugin API can be found here: [Figma Plugin API Documentation](https://developers.figma.com/docs/plugins/api/global-objects/).
