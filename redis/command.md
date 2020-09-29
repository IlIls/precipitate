# Redis

## Cluster

### Slot
#### Key管理
1. 计算key对应的Slot:
   ```cluster keyslot ${key}```
2. 获取指定Slot包含Key的数量:
   ```cluster countkeysinslot ${slot}```
3. 获知指定Slot中count个Key:
   ```cluster getkeysinslot ${slot} ${count}```

#### Slot迁移
1. 在目标节点执行命令，让目标节点准备导入Slot数据。 cluster setslot importing ${源节点ID}
2. 在源节点执行命令，让源节点准备迁出Slot数据。 cluster setslot migrating ${目标节点ID}
3. 在源节点执行命令，获取指定数量属于Slot的Key。 cluster getkeysinslot ${slot_id}
4. 在源节点执行命令，将第三步获取到的Key通过Pipeline迁移到目标节点。 migrate ${目标节点IP} ${目标节点PORT} "" ${destination_db} ${timeout_ms} REPLACE keys ${keys}，对于Redis集群，destination_db=0。
5. 执行命令，向集群广播，通知Slot分配给目标节点。 cluster setslot ${slot} NODE ${目标节点ID}
```
自动迁移脚本： move_slot.sh
```