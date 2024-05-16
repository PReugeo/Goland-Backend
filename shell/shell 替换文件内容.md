```shell
# 读取环境变量
KERNELSPEC=${KERNELSPEC_ARGS:-''}
DP_NAME=${DISPLAY_NAME:-''}
LANGUAGE=${LANGUAGE:-''}

if [[ -z "${KERNELSPEC}" ]] && [[ -z "${DP_NAME}" ]] && [[ -z "${LANGUAGE}" ]];
then
	echo "kernelspec not set, use default kernelspec"
else
	# kernel.json 
	kernelspec="{ \n\t \"language\": \"${LANGUAGE}\", \n\t \"display_name\": \"${DP_NAME}\", \n\t \"argv\": [ \n"
	# 分割 kernelspec arg 
	arr_kp=(${KERNELSPEC})
	for ((i=0;i<${#arr_kp[@]};i++));
	do
		end=$(echo $((${#arr_kp[@]}-1)))
		if [ "${i}" == ${end} ];
		then
			kernelspec=$kernelspec"\t\t\"${arr_kp[${i}]}\"\n"
		else
			kernelspec=$kernelspec"\t\t\"${arr_kp[${i}]}\",\n"
		fi
	done
	kernelspec=$kernelspec"\t ] \n }"
	# echo -e ${kernelspec}
	# 替换 /usr/local/share/jupyter/kernels/python3/kernel.json
	echo -e $kernelspec > /usr/local/share/jupyter/kernels/python3/kernel.json
fi
```
