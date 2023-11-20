# Harnessing the Power of DevContainers in Ruby and Jekyll Blog Projects

In terms of developer experience (DevEx), consistency and efficiency in our dev environments, especially when it comes to multiple developers are paramount. This is where [DevContainers](https://containers.dev/) come into play. For example, GitHub pages are based on Ruby and Jekyll. because Jekyll doesn't play well with Ruby versions 3.0 and above, you typically have to run it on Ruby versions that are 2.7+. This makes getting the developer environment setup a little bit tedious, as everyone has to manage multiple versions of Ruby in order to work with GitHub pages (you most likely already have an existing ruby version on your machine, especially if you're on Mac). DevContainers solves this issue by allowing you to have an isolated, consistent and reproducible environment across machines and developers. Here's some additional details about the advantages of DevContainers:

## Streamlined Development Environment

**Consistency Across the Board:** One of the major challenges in development is ensuring that the development environment is consistent across all team members' machines. DevContainers solve this by providing a Docker-based, containerized development environment. This means that whether you're on Windows, macOS, or Linux, the development experience remains uniform, eliminating the "it works on my machine" syndrome.

## Simplified Setup Process

**One-Click Setup:** Setting up a dev environment can be time-consuming, especially for new engineers. DevContainers encapsulate all the necessary dependencies, tools, and configurations in a single container. This means new team members or contributors can get up and running with a simple "clone and start" process, bypassing the often tedious environment setup.

## Isolated Development

**No More Dependency Hell:** {Ruby, Python, etc} projects often suffer from conflicting dependencies. DevContainers isolate your project in a container, ensuring that dependencies for one project don’t interfere with others. This isolation is particularly beneficial when working on multiple projects with different dependency requirements.

## Enhanced Productivity

**Tools at Your Fingertips:** DevContainers can be pre-configured with your preferred tools and extensions. For Ruby and Jekyll, this might include linters, syntax highlighters, and debuggers, all ready to go as soon as you start your container. This pre-packaged setup saves time and boosts productivity.

## Version Control Friendly

**Consistent Builds with devcontainer-lock.json:** The devcontainer.json file ensures that every member of your team is using the exact same build of the development environment. This consistency extends to your CI/CD pipelines, making sure that builds and tests run in an environment identical to the one on your local machine.

## Scalable and Collaborative

**Ease of Collaboration:** Sharing a DevContainer configuration makes it easier for other developers to contribute to your project. They can quickly spin up an identical environment, ensuring that code contributions are tested in the same context as your existing codebase.

## Continuous Integration Compatibility

**Seamless CI/CD Integration:** DevContainers align closely with the principles of CI/CD, particularly in terms of replicating a consistent environment for testing and deployment. This compatibility ensures that the environment where you develop is the environment where your code runs in production, reducing deployment surprises.

## Conclusion

Incorporating DevContainers into your DevEx streamlines your development process, fosters collaboration, and ensures consistency across all stages of development and deployment. By using this tool, you can focus more on writing code and less on the complexities of environment management. As a reference, the DevContainer setup I use to get an environment up and running for my GitHub pages setup can be found [here](https://github.com/tzehon/tzehon.github.io).
