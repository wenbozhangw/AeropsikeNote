## Planning a Deployment

为了在开发或生产环境中成功使用 Aerospike Server，请注意这些系统要求。

### Recommended Minimum Requirements

Aerospike Server 可在大多数主要的 64-bit Linux发行版上运行 - CentOS (Red Hat), Ubuntu, and Debian。它以适合安装环境的标准RPM和DEB软件包分布：
- CentOS 6 (RHEL 6), CentOS 7 (RHEL 7) & CentOS 8 (RHEL 8)
- Debian 8, 9 and 10
- Ubuntu 16.04, 18.04, and 20.04

为使Aerospike正常运行，建议的最低系统要求为：

- 64-bit Linux
- 2 GB of RAM

尽管建议最小为2 GB RAM（足以用于开发），但是您将需要确保有足够的RAM来存储要存储的数据量。

如果您打算进行生产部署，建议您阅读下面的部署计划指南。

### Deployment Planning
推荐的最低系统要求足以满足开发环境的要求，但不适用于生产环境。对于生产部署，请参阅以下步骤以确保成功部署生产环境。

1. 估算数据大小

在开始之前，请确定有关您要存储的以下基本问题的答案。要正确调整硬件大小，必须对数据进行粗略估计。
- 您将拥有几条记录？例如，您期望有多少个用户配置文件？
- 您计划在每个记录中存储多少数据？


Aerospike 允许您以后添加更多的存储，但是由于硬件相对便宜，因此从粗略估计所需的硬件资源开始是一个好主意。当前在Aerospike中存在单个设备和文件的限制。这些不得配置为大于2 TiB。

2. 确定存储类型

Aerospike允许您存储数据：
- In memory (without persistence)
- In memory (with persistence; data is backed up on disk)
- On flash (SSD) storage (with indexes in memory)

设置存储方法后，以后可能无法在不丢失数据的情况下更改该方法。

3. 确定命名空间的数量

Aerospike 数据库将数据存储在命名空间中。每个命名空间均根据其存储要求进行配置。许多Aerospike安装使用单个命名空间，所有数据都在同一存储介质中处理。但是，例如，如果要在内存中存储一组数据，而在SSD上存储另一组数据，则需要配置两个命名空间。

添加新的名称==命名空间需要完全重启集群。

4. 容量规划

有关如何适当调整部署大小（包括内存和存储要求）的帮助，请参阅 [Capacity Planning Guide](https://docs.aerospike.com/docs/operations/plan/capacity) 。如果不确定容量要求或只想尝试数据库，则可以跳过此步骤。

5. 服务器硬件

有关确定适当硬件的帮助，请参阅 [Server Hardware Requirements](https://docs.aerospike.com/docs/operations/plan/hardware) 。

6. 闪存存储

有关确定正确的闪存的帮助，请参阅 [Flash Storage Guide](https://docs.aerospike.com/docs/operations/plan/ssd) 。

7. 网络

关调整和配置网络的帮助，请参见 [Network Guide](https://docs.aerospike.com/docs/operations/plan/network) 。

Aerospike销售工程团队可帮助您调整硬件大小，并确定建议的硬件配置适合您的项目。请 [Contact us](https://docs.aerospike.com/contact/) 以获取有关调整硬件大小的帮助。

如果要安装Aerospike进行评估：

- 出于评估目的，您可能会发现使用DRAM运行测试数据库更为简单。根据您的SSD，设置过程可能需要一些时间。此外，我们不建议您过度配置作为笔记本电脑上虚拟驱动器的SSD。
- 请小心使用亚马逊评估性能。亚马逊是一个共享平台，网络延迟可能会因共享服务器的其他站点的流量而异。如果您只想使用Amazon进行评估，则应该知道Amazon不适合用于裸机部署。
- 如果使用VM进行测试，请注意，VM会增加任何存储（尤其是SSD）的延迟。结果，您将无法获得最佳性能。生产服务器通常不在VM内运行，因此这通常只是测试/评估时的问题。