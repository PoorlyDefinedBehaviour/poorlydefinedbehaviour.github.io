---
title: "How I write CI pipelines in 2023"
date: 2023-08-05T00:00:00-03:00
categories: ["ci", "rust", "dagger", "automation", "docker", "containers"]
draft: false
---

[Dagger] is a programmable CI/CD engine that runs your pipelines in containers[^what_is_dagger]. It runs your pipelines inside containers which makes it easier to test things locally. Not having to write yaml/bash/etc is a huge advantage for me.  

I'm working on a personal project that will allow people to deploy their code by selecting a [GitHub] repository. We have decided to use Dagger to clone the user provided [Git] repository and build a [Docker] image and it was extremely easy to get it working.

```rust
pub async fn build(repo_url: &str, branch: &str, image: &str) -> Result<String> {
    let client = dagger_sdk::connect()
        .await
        .map_err(|err| anyhow!("connecting to dagger: {}", err.to_string()))?;

    let git_ref = client
        .git(repo_url)
        .branch(branch);

    let container_registry_secret = client.set_secret("CONTAINER_REGISTRY_SECRET", "TODO");
    let registry_secret_id = container_registry_secret
        .id()
        .await
        .map_err(|err| anyhow!("fetching secret id: {}", err.to_string()))?;

    let directory_id = git_ref
        .tree()
        .id()
        .await
        .map_err(|err| anyhow!("fetching git tree directory id: {}", err.to_string()))?;

    let image_ref = client
        .container()
        .from("node:20-alpine")
        .with_directory("/app", directory_id)
        .with_workdir("/app")
        .with_exec(vec!["npm", "install", "--only=production"])
        .with_entrypoint(vec!["npm", "start"])
        .with_registry_auth("docker.io", "poorlydefinedbehaviour", registry_secret_id)
        .publish(image)
        .await
        .map_err(|err| anyhow!("publishing container image: {}", err.to_string()))?;

    info!("pushed image to container registry: {image_ref}");

    Ok(image)
}
```

The container building part will get more complex as we add more functionality and Dagger will certainly become more appreciated.  

Since we are dealing with user provided code, we cannot trust that the provided code will not try to break the system. We are planning to run containers that contain untrusted code inside sandboxes in [Kubernetes] using [kata-containers] and a virtual machine monitor like AWS's [Firecracker].

[Dagger]: https://dagger.io/  
[Github]: https://github.com/  
[Git]: https://git-scm.com/  
[Docker]: https://www.docker.com/  
[kata-containers]: https://github.com/kata-containers  
[Firecracker]: https://firecracker-microvm.github.io/  
[Kubernetes]: https://kubernetes.io/  
[^what_is_dagger]: https://docs.dagger.io/#what-is-dagger/  