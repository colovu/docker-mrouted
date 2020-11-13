# Ver: 1.3 by Endial Fang (endial@126.com)
#

# 预处理 =========================================================================
FROM colovu/abuilder as builder

# sources.list 可使用版本：default / tencent / ustc / aliyun / huawei
ARG apt_source=default

# 编译镜像时指定用于加速的本地服务器地址
ARG local_url=""

ENV APP_NAME=mrouted \
	APP_VERSION=4.1

RUN select_source ${apt_source};
RUN install_pkg ipset-dev libnftnl-dev iptables-dev libnfnetlink-dev openssl-dev libnl3-dev libffi-dev

# 下载并解压软件包
RUN set -eux; \
	appName="${APP_NAME}-${APP_VERSION}.tar.gz"; \
	[ ! -z ${local_url} ] && localURL=${local_url}/mrouted; \
	appUrls="${localURL:-} \
		https://ftp.troglobit.com/mrouted \
		"; \
	download_pkg unpack ${appName} "${appUrls}"; 

# 源码编译软件包
RUN set -eux; \
# 源码编译方式安装: 编译后将原始配置文件拷贝至 ${APP_DEF_DIR} 中
	APP_SRC="/usr/local/${APP_NAME}-${APP_VERSION}"; \
	cd ${APP_SRC}; \
	./configure \
		--prefix=/usr/local/${APP_NAME} \
		--runstatedir=/var/run/${APP_NAME}; \
	make -j "$(nproc)"; \
	make install; 

# 检测并生成依赖文件记录
RUN set -eux; \
	scanelf --needed --nobanner --format '%n#p' --recursive /usr/local/${APP_NAME} | \
		tr ',' '\n' | \
		sort -u | \
		awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } \
		{ print "so:" $1 }' >/usr/local/${APP_NAME}/runDeps;

# 镜像生成 ========================================================================
FROM colovu/alpine:3.12

ARG apt_source=default
ARG local_url=""

ENV APP_NAME=mrouted \
	APP_USER=mroute \
	APP_EXEC=run.sh \
	APP_VERSION=4.1

ENV	APP_HOME_DIR=/usr/local/${APP_NAME} \
	APP_DEF_DIR=/etc/${APP_NAME}

ENV PATH="${APP_HOME_DIR}/sbin:${PATH}" \
	LD_LIBRARY_PATH="${APP_HOME_DIR}/lib"

LABEL \
	"Version"="v${APP_VERSION}" \
	"Description"="Docker image for ${APP_NAME}(v${APP_VERSION})." \
	"Dockerfile"="https://github.com/colovu/docker-${APP_NAME}" \
	"Vendor"="Endial Fang (endial@126.com)"

# 选择软件包源
RUN select_source ${apt_source}
RUN install_pkg iproute2

# 从预处理过程中拷贝软件包(Optional)
COPY --from=builder /usr/local/mrouted/ /usr/local/mrouted

# 安装依赖软件包
RUN install_pkg `cat ${APP_HOME_DIR}/runDeps`; 

COPY customer /
RUN create_user && prepare_env

# 执行预处理脚本，并验证安装的软件包
RUN set -eux; \
	override_file="/usr/local/overrides/overrides-${APP_VERSION}.sh"; \
	[ -e "${override_file}" ] && /bin/bash "${override_file}"; \
	mrouted --version ;

# 默认提供的数据卷
VOLUME ["/srv/conf", "/var/log"]

# 默认使用gosu切换为新建用户启动，必须保证端口在1024之上
# EXPOSE 8080

# 容器初始化命令，默认存放在：/usr/local/bin/entry.sh
ENTRYPOINT ["entry.sh"]

# 应用程序的服务命令，必须使用非守护进程方式运行。如果使用变量，则该变量必须在运行环境中存在（ENV可以获取）
CMD ["${APP_EXEC}"]

