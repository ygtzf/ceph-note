up acting 那两个set在通常情况下是一致的
当一个pg副本数达不到pool size时，这两个set会变化
up set是crush理论算的
acting set就像字面意思， 是临时的
acting set表示当前实际的pg分布是这样的
等最后数据rebalance recover完成后，两个又一致了
acting set是pg_temp的set
