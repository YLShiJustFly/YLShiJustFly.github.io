## Preface: Dominating the IO500 Rankings
Having seen the DAOS project dominating the IO500 rankings, I've been keeping an eye on this project (though not diving deep into it).
![image.png|IO500](../../image/io500.png)
## Background
In previous articles, I briefly introduced the DAOS distributed storage project.
However, with Intel terminating the Optane business in 2022, many people began to wonder: **Can DAOS continue after losing its "core hardware support"? Where is its future?**
## Short-term Impact, but Not the End
The discontinuation of Optane did have a significant impact on DAOS, especially in metadata acceleration and persistence. However, DAOS was not trapped by this "hardware earthquake" but instead achieved "rebirth" through architectural adjustments:
- **Introduction of Write-Ahead Logging (WAL)**: Writing logs to DRAM and syncing to NVMe SSDs to replace the low-latency persistence path that originally relied on PMem
- **Continuous optimization of NVMe access paths**: Maintaining high IOPS and low latency even in environments without Optane
![image.png|IO500](../../image/daos01.png)
## The Power of Open Source Community: From Intel Dominance to Foundation Governance
In November 2023, the **DAOS Foundation** was officially established, jointly supported by Argonne National Laboratory, Enakta Labs, Google, HPE, and Intel. The foundation has a Technical Steering Committee, chaired by Johann Lombardi, a senior technical expert at HPE and long-term DAOS developer and advocate.
This transformation means:
- DAOS transitioned from an Intel-dominated project to a neutral open source ecosystem
- Attracted a broader range of users and contributors, making the development roadmap more open and transparent
- Enhanced confidence in continued future evolution
### Open Source Metrics: DAOS vs. Ceph
![image.png|IO500](../../image/star.png)
![image.png|IO500](../../image/company.png)
## Looking Forward: Version 2.8 and 3.0 Roadmap
**DAOS v2.6** was released in July 2024, marking the last Intel release.
According to the latest release information:
- **DAOS 2.8** (end of 2025): This is the first release version transitioning from Intel dominance to community leadership, continuing to enhance NVMe adaptation, metadata engine, and K8s cloud-native support
- **DAOS 3.0** (expected 2026): Will bring more thorough modular design, strengthening integration with AI and container ecosystems
![image.png|IO500](../../image/roadmap.png)
## Typical DAOS User Cases
### 1. Aurora Supercomputer (U.S. Department of Energy Argonne National Laboratory)
- **Deployment Scale**: Aurora is an exascale supercomputer supported by the U.S. Department of Energy, with DAOS used as one of its primary storage systems
- **Application Scenarios**: Supporting large-scale AI model training, scientific simulation, and data analysis tasks
- **Technical Highlights**: DAOS provides high IOPS and low latency performance, meeting Aurora's strict requirements for high-performance storage
### 2. SuperMUC-NG Supercomputer (Leibniz Supercomputing Centre, Germany)
- **Deployment Scale**: SuperMUC-NG is one of Germany's leading supercomputing platforms, with DAOS integrated for high-performance computing tasks
- **Application Scenarios**: Supporting scientific research and engineering simulation high-performance computing needs
- **Technical Highlights**: DAOS's high concurrent access capability and scalability make it an ideal storage solution for SuperMUC-NG
### 3. Google Cloud
- **Application Scenarios**: Parallelstore is suitable for various high-performance computing and AI training scenarios, with DAOS as the underlying technology
- **Technical Highlights** (at approximately 100TB deployment scale):
  - **Throughput**: Up to ~115 GiB/s
  - **IOPS**: Read approximately 3 million/second, write approximately 1 million/second
  - **Latency**: Approximately 0.3 milliseconds
## Current Challenges and Limitations of DAOS
DAOS still runs on multiple top-tier supercomputing platforms, such as Argonne's Aurora system and Germany's Leibniz Supercomputing Centre's SuperMUC-NG. However, DAOS also faces new challenges:
### No Support for NVIDIA GPUDirect
In the AI training and inference domain, NVIDIA GPU servers dominate, and most storage hardware and software vendors support the GPUDirect protocol. However, DAOS currently does not support it, creating a gap compared to competitors like WEKA in AI training/inference scenarios. However, DAOS does support RDMA.
### Insufficient Support for Modern Deployment Models
Support for containers, Kubernetes, and other modern deployment models still needs strengthening.
### Relatively Small User Base
Compared to well-known open source software like Ceph and Lustre, DAOS's current users are mainly research institutions and HPC users. The user scale is not large enough, and the industries represented by users are not diverse enough.
## DAOS Resources Summary
- **[DAOS WIKI](https://daos-stack.github.io/)**: Very comprehensive content
- **[DAOS Slack](https://daos-stack.slack.com/)**: Community communication platform
- **[DAOS Official Website](https://daos.io/)**: Official website
- **[DAOS Github](https://github.com/daos-stack/daos)**: Source code repository
## Conclusion
DAOS has not declined due to Optane's termination; instead, it has accelerated its independence through this "weaning." Through technical evolution and community support, it is gradually breaking free from dependence on specific hardware and moving toward a more universal and robust development path.
It must be said that DAOS is still a very young project with many shortcomings, such as insufficient user scale and limited functionality. However, compared to systems like Ceph, DAOS's architectural and performance advantages are still quite obvious.
How will DAOS develop in the future? We'll just have to wait and see...
## References
1. [DAOS WIKI](https://daos-stack.github.io/)
2. [DAOS Slack](https://daos-stack.slack.com/)
3. [DAOS Official Website](https://daos.io/)
4. [DAOS Github](https://github.com/daos-stack/daos)
5. [DAOS file system lives on after Optane](https://zhuanlan.zhihu.com/p/1910394685379282773) 
