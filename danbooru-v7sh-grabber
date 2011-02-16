#!/bin/sh
# LICENSE: WTFPLv2
#                               (c) Aleksander Tumin <itakingiteasy@gmail.com>


# i use `` instead of $() because solaris does not support $()'s

# i also used some other thing only because it was compatible with solaris,
# but i forgot which one

# 1.  Script scheme:
# 1.1 <Core Functions Section>
# 1.2 <Init Section>
# 1.3 <Options Parse Section>
# 1.4 <Variable Validation Checks>
# 2.  Engines:
# 2.1 <Danbooru Engine>
# 2.2 <Gelbooru Engine>
# 3.  Main:
# 3.1 <Main Section>

# Main Data flow:

# 001. First, functions in <Core Functions Section> are defined.
# There are nine of them and all must be defined _before_ <Init Section>
# 002. Then starts <Init Section>, where global variables are defined
# 003. After defining default values of global variables, 
# <Options Parse Section> starts, re-defining global variables by user input
# command-line arguments and setting ${s_tag_list} variable with specified tags.
# ${l_mode} set to 'download' or 'search'.
# ${l_engine} set to 'danbooru' or 'gelbooru'.
# 004. After patsing command-line options sanity check are executing, testing
# validity of specified values
# 005. Engine query() function is defined. More on that later.
# 006. process() function is defined.
# case "${l_mode}" in 
#	  "search") process() for search is defined ;;
#	"download") process() for download is defined ;;
# esac
# 007. query() function is defined
# case "${l_engine}" in
# 	"danbooru")
#		case "${l_mode}" in
#			  "search") query() for danbooru search is defined ;;
#			"download") query() for danbooru download is defined ;;
#		esac
#        ;;
# 	"gelbooru")
#		case "${l_mode}" in
#			  "search") query() for gelbooru search is defined ;;
#			"download") query() for gelbooru download is defined ;;
#		esac
#        ;;
# esac
# 008. ${s_tag_list} splitting to ${tag_group}'s by ',' (coma) symbol.
# 009. For each ${tag_group} `process "${tag_group}"` is executed
# 010. process() calling query() function and parses it's output


# ############################ Core Functions Section ######################## #

log() {
(
	type="$1"
	shift
	format="$1"
	shift
	if printf "%s\n" "${s_verbose}" | 
		grep -v -- "`printf "%s\n" "${type}" | 
			sed 's/_part//'`" >/dev/null; then
		return 1
	fi
	case "${type}" in
		"notice") printf "%s ${format}" "[Notice]" "$@" 1>&2; ;;
		"notice_part") printf -- "${format}" "$@" 1>&2; ;;
		"error") printf "%s ${format}" "[Error]" "$@" 1>&2; ;;
		"error_part") printf -- "${format}" "$@" 1>&2; ;;
		"debug") printf "%s ${format}" "[Debug]" "$@" 1>&2; ;;
		"debug_part") printf -- "${format}" "$@" 1>&2; ;;
		"message") printf -- "${format}" "$@" 1>&2 ;;
		"export") printf -- "${format}" "$@"; ;;
	esac
)
}


get_binary() {
(
	for binary in "$@"; do
		if command -v -- "${binary}" > /dev/null 2>&1; then
			printf "%s\n" "${binary}"
			break
		fi
	done
)
}

get_single_opt() {
(
	tmp_is_argument="false"
	long_option="$1"
	shift
	short_option="$1"
	shift
	for opt in "$@"; do
		if [ "${long_option}" = "${opt}" ] || [ "${short_option}" = "${opt}" ]; then
			tmp_is_argument="true"
			continue
		fi
		if [ "false" = "${tmp_is_argument}" ]; then
			continue
		fi
		printf "%s\n" "${opt}"
		break
	done
)
}

cleanup() { 
(
	tmpdir="$1"
	shift
	log "notice" "%s" "Cleaning up..."
	rm -rf -- "${tmpdir}"
	log "notice_part" "%s\n" "done"
)
}

downloader() {
(
	remote_file_url="$1"
	shift
	local_file="$1"
	shift
	url_host="`printf "%s\n" "${remote_file_url}" | sed 's|^.*://||;s|/.*||'`"
	url_path="`printf "%s\n" "${remote_file_url}" | sed 's/:[0-9]*$//;s|.*://[^/]*[/]\{0,1\}$|/|'`"
	url_port="`printf "%s\n" "${remote_file_url}" | sed -n 's/.*:\([0-9]*\)$/\1/p'`"
	url_port="${url_port:-80}"

	case "${b_downloader}" in
		*"wget")
			wget -q -c "${remote_file_url}" -O "${local_file}"
		;;
		*"curl")
			curl -s -C - -o "${local_file}" -- "${remote_file_url}"
		;;
		*"axel")
			axel -q -o "${local_file}" -- "${remote_file_url}"
		;;
		*"fetch")
			fetch -m -o- -- "${remote_file_url}" > "${local_file}" 2>/dev/null
		;;
	esac
)
}

urlencode() {
(
	input="$@"
	printf "%s" "${input}" | 
		od -t x1 -v | 
		sed 's/^[0-9]*//g;s/[ ]\{1,\}/%/g;s/[%]*$//g;' | 
		while read line; do 
			printf "%s" "${line}";
		done
		printf "\n"
)
}

tags_urlencode() {
(
	tags="$@"
	out=""
	# no "" !
	for tag in ${tags}; do
		is_ored="false"
		encoded_tag="`urlencode "${tag}"`"
		out="${out}+${encoded_tag}"
	done
	printf "%s\n" "${out}" | sed 's/^+//'
)
}

hasher() {
(
	case "${b_hasher}" in
		*"sha1sum")
			sha1sum | awk '{print $1}'
		;;
		*"sha1")
			sha1
		;;
		*"digest")
			digest -a sha1
		;;
	esac
)
}

password_hash() {
(
	password="$1"
	shift
	printf -- "${password}" | hasher | awk '{print $1}'
)
}

print_help() {
(
	log "message" "%s\n" "\
USAGE: $0 [OPTIONS] <tagA1[ tagA2 ...][, tagB1 ...]>
	-h	--help			This help
	-v 	--version		Script Version
	-d	--download		Download tags instead of searching them
	-e	--engine		Danbooru engine, danbooru or gelbooru
	-u	--username		Your danbooru usrname
	-p	--password		Your danbooru password
	-du	--danbooru-url		Danbooru url
	-i	--interface		Interface, 'xml' or 'json' for danbooru
					'xml' for gelbooru
	-so	--search-order		Search order, 'count', 'name' or 'date'
	-sl	--search-limit		Limit search results. 0 - unlimited
	-dl	--download-limit	Download page size. Greater then 1
	-dp	--download-page		Download page offset
	-dm	--download-mode		Download mode, 'onedir' or 'export'
	-dfn	--download-file-name	Filenaming pattern. Variables are:
					post:height - height of the picture
					post:width - width of the picture
					post:tags - tags of the picture
					post:md5 - md5 of the picture
					post:id - id of the picture
	-dwd	--download-working-directory	Working directory"
)
}

# ############################################################################ #


# ################################## Init Section ############################ #

# workaround for solaris
PATH="${PATH}:/usr/sfw/bin"
export PATH

# global
g_version="Danbooru v7sh grabber v0.20.19 for Danbooru API v1.13.0"
# strings
s_tag_list=""
s_verbose="`get_single_opt "--verbose" "-v" "$@"`"
s_verbose="${s_verbose:-notice message error export}"
s_not_found_binaries=""
s_file_name_format="post:md5"
s_username=""
s_password=""
# logic
l_mode="search"
l_engine="danbooru"
l_interface="xml"
l_search_order="count"
l_search_limit="0"
l_download_limit="100"
l_download_mode="onedir" # currentdir, many
l_page="1"
l_used_binaries_list="[ printf cat grep sed od dd awk"
l_supported_downloaders_list="`get_single_opt "--binary-downloader" "-bd" "$@"` wget curl axel fetch"
l_supported_sha1_hashers_list="digest sha1 sha1sum"
#binaries
b_downloader="`get_binary ${l_supported_downloaders_list}`"
b_hasher="`get_binary ${l_supported_sha1_hashers_list}`"
#paths
p_tempdir="/tmp/danbooru_v7sh_grabber_$$"
p_temp_query="${p_tempdir}/query"
p_temp_image="${p_tempdir}/image"
p_danbooru_url=""
p_working_directory="./"

if [ ! "${b_downloader}" ]; then
	log "error" "%s\n" "No suitable downloader found. Consider \
putting to your PATH directories one of folowing binaries: \
`printf "%s\n" ${l_supported_downloaders_list} | tr ' ' '\n' | sort -u | tr '\n' ' '`"
	exit 1
fi

log "notice" "%s\n" "Downloader found: ${b_downloader}"

for binary in ${l_used_binaries_list}; do
	if [ ! "`command -v "${binary}"`" ]; then
		s_not_found_binaries="${binary} ${s_not_found_binaries}"
	fi	
done

if [ ! -z "${s_not_found_binaries}" ]; then
	log "error" "%s\n" "Folowing required binaries was not found \
in your PATH directories: ${s_not_found_binaries}"
	exit 1
fi

trap "cleanup ${p_tempdir}" 0
mkdir "${p_tempdir}"

# ############################################################################ #


# ############################ Options Parse Section ######################### #

if [ -z "$1" ]; then
	set -- "--help"
fi

while [ ! -z "$1" ]; do
	case "$1" in
		 "-h" | "--help")
			print_help
			exit 0
		;;		
		 "-V" | "--version")
			log "message" "%s: %s\n" "$0" "${g_version}"
			exit 0
		;;
		 "-v" | "--verbose")
		 	[ ! -z "$2" ] && shift
		;;
		 "-bd" | "--binary-downloader")
		 	[ ! -z "$2" ] && shift
		;;
		 "-d" | "--download")
			l_mode="download"
		;;
		 "-e" | "--engine")
			l_engine="$2"
		 	[ ! -z "$2" ] && shift
		;;
		"-u" | "--username")
			s_username="$2"
		 	[ ! -z "$2" ] && shift
		;;
		"-p" | "--password")
			s_password="$2"
		 	[ ! -z "$2" ] && shift
		;;
		 "-du" | "--danbooru-url")
			p_danbooru_url="$2"
		 	[ ! -z "$2" ] && shift
		;;
		 "-i" | "--interface")
			l_interface="$2"
		 	[ ! -z "$2" ] && shift
		;;
		 "-so" | "--search-order")
			l_search_order="$2"
		 	[ ! -z "$2" ] && shift
		;;
		 "-sl" | "--search-limit")
			l_search_limit=`printf "%d\n" $2` || {
				log "error" "%s\n" "Invalid search limit: \
'$2'. Must be a number."
				exit 1
			}
		 	[ ! -z "$2" ] && shift
		;;
		 "-dl" | "--download-limit")
			l_download_limit=`printf "%d\n" $2` || {
				log "error" "%s\n" "Invalid download limit: \
'$2'. Must be a number."
				exit 1
			}
		 	[ ! -z "$2" ] && shift
		;;
		 "-dp" | "--download-page")
			l_page=`printf "%d\n" $2` || {
				log "error" "%s\n" "Invalid starting page \
number: '$2'. Must be a number."
				exit 1
			}
		 	[ ! -z "$2" ] && shift
		;;
		 "-dm" | "--download-mode")
			l_download_mode="$2"
		 	[ ! -z "$2" ] && shift
		;;
		"-dfn" | "--download-file-name")
			s_file_name_format="$2"
			[ ! -z "$2" ] && shift
		;;
		"-dwd" | "--download-working-directory")
			p_working_directory="$2"
			[ ! -z "$2" ] && shift
		;;
		*)
			s_tag_list="${s_tag_list} $1"
		;;
	esac
	shift
done

s_tag_list="`printf "%s\n" "${s_tag_list}" | sed 's/^[ ]*//g;s/[ ]*$//g;s/[ ]\{1,\}/ /g;'`"



# ############################################################################ #


# ########################### Variable Validation Checks ##################### #

if [ ! -z "${s_password}" ] && [ -z "${s_username}" ]; then
	log "error" "%s\n" "Username must be specified"
	exit 1
fi

if [ -z "${s_password}" ] && [ ! -z "${s_username}" ]; then
	log "error" "%s\n" "Password must be specified"
	exit 1
fi

case "${l_engine}" in
	"danbooru")
		if [ -z "${p_danbooru_url}" ]; then
			p_danbooru_url="http://danbooru.donmai.us"
		fi
		c_anonymous_tag_limit="2"
		c_registred_tag_limit="6"
		if [ -z "${s_username}" ]; then
			IFS=","
			for tag_group in ${s_tag_list}; do
				if [ `printf "%s\n" "${tag_group}" | wc -w | awk '{print $1}'` -gt ${c_anonymous_tag_limit} ]; then
					log "error" "%s\n" "You can't download more \
then two tags anonymously with danbooru engine."
				exit 1
				fi
			done
			IFS=" "
		else
			if [ -z "${b_hasher}" ]; then
					log "error" "%s\n" "No suitable password \
hasher found. Consider putting into your PATH directories one of folowing \
binaries: ${l_supported_sha1_hashers_list}"
					exit 1
			fi
			IFS=","
			for tag_group in ${s_tag_list}; do
				if [ `printf "%s\n" "${tag_group}" | wc -w | awk '{print $1}'` -gt ${c_registred_tag_limit} ]; then
					log "error" "%s\n" "You can't download more \
then six tags with danbooru engine."
				exit 1
				fi
			done
			IFS=" "
			s_password="`password_hash "choujin-steiner--${s_password}--"`"
		fi
		case "${l_interface}" in
			"xml") ;;
			"json") ;;
			*)
				log "error" "%s\n" "Invalid interface: \
'${l_interface}'. Must be either 'xml' or 'json' for danbooru engine."
				exit 1
			;;
		esac
	;;
	"gelbooru")
		if [ -z "${p_danbooru_url}" ]; then
			p_danbooru_url="http://gelbooru.com"
		fi
		c_anonymous_tag_limit="-1"
		c_registred_tag_limit="-1"
		case "${l_interface}" in
			"xml") ;;
			*)
				log "error" "%s\n" "Invalid interface: \
'${l_interface}'. Must be 'xml' for gelbooru engine."
				exit 1
			;;
		esac
	;;
	*)
		log "error" "%s\n" "Invalid engine: '${l_engine}'. Must be \
either 'danbooru' or 'gelbooru'"
		exit 1
	;;
esac

case "${l_search_order}" in
	"date") ;;
	"count") ;;
	"name") ;;
	*)
		log "error" "%s\n" "Invalid search order: '${l_search_order}'. \
Must be either 'date', 'count' or 'name'."
		exit 1
	;;
esac

case "${l_download_mode}" in
	"onedir") ;;
	"export") ;;
	*)
		log "error" "%s\n" "Invalid download mode: \
'${l_download_mode}'. Must be 'onedir' or 'export'."
		exit 1
	;;
esac

if [ ${l_search_limit} -lt 0 ]; then
	log "error" "%s\n" "Invalid search limit number: \
${l_search_limit}. Must be greater or equal to zero."
	exit 1
fi

if [ ${l_download_limit} -lt 1 ]; then
	log "error" "%s\n" "Invalid download limit number: \
${l_download_limit}. Must be greater then zero."
	exit 1
fi

if [ ${l_page} -lt 1 ]; then
	log "error" "%s\n" "Invalid page number: ${l_download_limit}. Must be \
greater then zero."
	exit 1
fi

if [ ! -d "${p_working_directory}" ]; then
	log "error" "%s\n" "Invalid working directory: ${p_working_directory}"
	exit 1
fi


# ############################################################################ #


# ################################ Danbooru Engine ########################### #

engine_danbooru() {
	mode="$1"
	shift
	api_interface="$1"
	shift
	login_string=""
	if [ ! -z "${s_username}" ]; then
		login_string="&login=${s_username}&password_hash=${s_password}"
	fi
	case "${mode}" in
		"search")
			case "${api_interface}" in
				"xml")
					parse() {
					(
						grep '<tag ' | while read line; do
							printf "%s\n" "${line}" | 
							sed  's/.*type="\([0-9]*\)".*/\1/g;' | 
							tr '\n' ' '
							printf "%s\n" "${line}" | 
							sed  's/.*count="\([0-9]*\)".*/\1/g;' | 
							tr '\n' ' '
							printf "%s\n" "${line}" | 
							sed  's/.*name=\"\([^\"]*\)\".*/\1/g;'
						done
					)
					}
					parse_deep() {
					(
						name="$1"
						shift
						out=`sed -n '/^.*<posts[^<]*count="\([0-9]*\)"[^>]*>.*$/{
							s//mixed \1 /;
							p;
							}'`
						if [ ! -z "${out}" ]; then
							printf "%s" "${out}"
							printf "%s\n" "${name}"
						fi
					)
					}
				;;
				"json")
					parse() {
					(
						sed 's/^\[{//g;s/}\]$//g;' |
						sed 's/},{/\
/g;/^\[\]$/d'					| while read line; do
							printf "%s\n" "${line}" | 
							sed 's/.*"type":\([0-9]*\).*/\1/g;' | 
							tr '\n' ' '
					
							printf "%s\n" "${line}" | 
							sed 's/.*"count":\([0-9]*\).*/\1/g;' | 
							tr '\n' ' '
					
							printf "%s\n" "${line}"   |
							sed 's/.*"name":"\([^,]*\)".*/\1/g;'
						done
					)
					}
					parse_deep() {
					(
						log "error" "%s\n" "Sorry, searching for more \
then one tag is not possible with json interface. Falling back to xml one."
						exit 1
					)
					}
				;;
			esac
			type_format() {
				sed '	s/^0/general/;
					s/^1/artist/;
					s/^2/unknown/;
					s/^3/copyright/;
					s/^4/character/;'
			}
			query() {
			(
				query_type="$1"
				shift
				url_encoded_tags="$1"
				shift
				orderby="$1"
				shift
				limit="$1"
				shift
				if [ "simple" = "${query_type}" ]; then
					downloader "${p_danbooru_url}/tag/index.${api_interface}?name=${url_encoded_tags}&order=${orderby}&limit=${limit}${login_string}" "${p_temp_query}"
					printf "\n" >> "${p_temp_query}"
					cat -- "${p_temp_query}" | parse | type_format
					rm -- "${p_temp_query}"
				else
					downloader "${p_danbooru_url}/post/index.xml?tags=${url_encoded_tags}&limit=1${login_string}" "${p_temp_query}"
					out="`cat -- "${p_temp_query}" | parse_deep "${tag_group}"`"
					result="$?"
					rm -- "${p_temp_query}"
					if [ "${result}" -eq 1 ]; then	
						engine_danbooru "search" "xml"
						query "${url_encoded_tags}" "${orderby}" "${limit}"
					else
						 printf "%s\n" "${out}" | type_format
					fi
				fi
			)
			}
		;;
		"download")
			case "${api_interface}" in
				"xml")
					parse() {
					(
						sed -n '/.*<post [ ]*/{s///;s|/>$||;p;}' |
						while read line; do
							post_rating="`printf "%s\n" "${line}" | 
							sed 's/.*rating=\"\([^\"]*\)\".*/\1/g;
							s/e/explicit/g;s/s/safe/g;
							s/q/questionable/g;'`"
							post_height="`printf "%s\n" "${line}" | 
							sed 's/.*height=\"\([^\"]*\)\".*/\1/g;'`"
							post_width="`printf "%s\n" "${line}" | 
							sed 's/.*width=\"\([^\"]*\)\".*/\1/g;'`"
							post_tags="`printf "%s\n" "${line}" | 
							sed 's/.*tags=\"\([^\"]*\)\".*/\1/g;s/ /-/g;' | dd bs=250 count=1 2>/dev/null`"
							post_md5="`printf "%s\n" "${line}" | 
							sed 's/.*md5=\"\([^\"]*\)\".*/\1/g;'`"
							post_url="`printf "%s\n" "${line}" | 
							sed 's/.*file_url=\"\([^\"]*\)\".*/\1/g;'`"
							post_id="`printf "%s\n" "${line}" | 
							sed 's/.*id=\"\([^\"]*\)\".*/\1/g;'`"
							format_name | sed 's/&gt;/>/g;
							s/&lt;/</g;s/&quot;/"/g;s/&amp;/\&/g;'
						done
					)
					}
				;;
				"json")
					parse() {
					(
						sed 's/^\[{//g;s/}\]$//g;' |
						sed 's/},{/\
/g;/^\[\]$/d'					| while read line; do
							post_rating="`printf "%s\n" "${line}" | 
							sed 's/.*\"rating\":\"\([^\"]*\)\".*/\1/g;
							s/e/explicit/g;s/s/safe/g;
							s/q/questionable/g;'`"
							post_height="`printf "%s\n" "${line}" |
							sed 's/.*\"height\":\([0-9]*\).*/\1/g'`"
							post_width="`printf "%s\n" "${line}" |
							sed 's/.*\"width\":\([0-9]*\).*/\1/g'`"
							post_tags="`printf "%s\n" "${line}" | 
							sed 's/.*\"tags\":\"\([^,]*\)\".*/\1/g' | dd bs=250 count=1 2>/dev/null`"
							post_md5="`printf "%s\n" "${line}" | 
							sed 's/.*\"md5\":\"\([0-9a-z]*\)\".*/\1/g;'`"
							post_url="`printf "%s\n" "${line}" |
							sed 's/.*\"file_url\":\"\([^,]*\)\".*/\1/g'`"
							post_id="`printf "%s\n" "${line}" |
							sed 's/.*\"id\":\([0-9]*\).*/\1/g'`"
							format_name | sed 's/u003E/>/g;s/u003C/</g;s/u0026/\&/g;'
						done
					)
					}
				;;
			esac
			get_count() {
			(
				url_encoded_tags="$1"
				shift
				downloader "${p_danbooru_url}/post/index.xml?tags=${url_encoded_tags}&limit=1${login_string}" "${p_temp_query}"
				cat -- "${p_temp_query}" | sed -n '/^.*<posts count="\([0-9]*\)" offset="[0-9]*">.*$/{s//\1/;p;}'
				rm -- "${p_temp_query}"
			)
			}
			format_name() {
			(
				post_ext="`printf "%s\n" "${post_url}" | 
				sed 's/.*\(\.[^.]*\)$/\1/;'`"
				post_ext_length="`printf "%s" "${post_ext}" | 
				wc -c | awk '{print $1}'`"
				name_length="`expr 250 - "${post_ext_length}"`"

				# no "" !
				sed_string=`printf "%s\n" "s,post:rating,${post_rating},g;
				s,post:height,${post_height},g;
				s,post:width,${post_width},g;
				s,post:tags,${post_tags},g;
				s,post:md5,${post_md5},g;
				s,post:id,${post_id},g;" | sed 's/&/\\\&/g;'`

				name="`printf "%s\n" "${s_file_name_format}" | 
				sed -- "${sed_string}" |
				dd bs="${name_length}" count=1 2>/dev/null`"
				printf "%s %s\n" "${post_url}" "${name}${post_ext}"
			)
			}
			query() {
			(
				url_encoded_tags="$1"
				shift
				page="$1"
				shift
				limit="$1"
				shift
				downloader "${p_danbooru_url}/post/index.${api_interface}?tags=${url_encoded_tags}&page=${page}&limit=${limit}${login_string}" "${p_temp_query}"
				printf "\n" >> "${p_temp_query}"
				cat -- "${p_temp_query}" | parse
				rm -- "${p_temp_query}"
			)
			}
		;;
	esac
}

# ############################################################################ #


# ################################ Gelbooru Engine ########################### #

engine_gelbooru() {
	mode="$1"
	shift
	api_interface="$1"
	shift
	case "${mode}" in
		"search")
			case "${api_interface}" in
				"xml")
					parse_deep() {
					(
						name="$1"
						shift
						out="`sed -n '/^.*<posts[^<]*count="\([0-9]*\)"[^>]*>.*$/{
							s//mixed \1 /;
							p;
							}'`"
						if [ ! -z "${out}" ]; then
							printf "%s" "${out}"
							printf "%s\n" "${name}"
						fi
					)
					}
				;;
			esac
			query() {
				query_type="$1"
				shift
				url_encoded_tags="$1"
				shift
				orderby="$1"
				shift
				limit="$1"
				shift
				downloader "${p_danbooru_url}/index.php?page=dapi&s=post&q=index&tags=${url_encoded_tags}&limit=${limit}" "${p_temp_query}"
				printf "\n" >> "${p_temp_query}"
				cat -- "${p_temp_query}" | parse_deep "${tag_group}"
				rm -- "${p_temp_query}" 
			}
		;;
		"download")
			case "${api_interface}" in
				"xml")
					parse() {
					(
						sed -n '/.*<post [ ]*/{s///;s|/>$||;p;}' |
						while read line; do
							post_rating="`printf "%s\n" "${line}" | 
							sed 's/.*rating=\"\([^\"]*\)\".*/\1/g;
							s/e/explicit/g;s/s/safe/g;
							s/q/questionable/g;'`"
							post_height="`printf "%s\n" "${line}" | 
							sed 's/.*height=\"\([^\"]*\)\".*/\1/g;'`"
							post_width="`printf "%s\n" "${line}" | 
							sed 's/.*width=\"\([^\"]*\)\".*/\1/g;'`"
							post_tags="`printf "%s\n" "${line}" | 
							sed 's/.*tags=\"\([^\"]*\)\".*/\1/g;s/ /-/g;' | dd bs=250 count=1 2>/dev/null`"
							post_md5="`printf "%s\n" "${line}" | 
							sed 's/.*md5=\"\([^\"]*\)\".*/\1/g;'`"
							post_url="`printf "%s\n" "${line}" | 
							sed 's/.*file_url=\"\([^\"]*\)\".*/\1/g;'`"
							post_id="`printf "%s\n" "${line}" | 
							sed 's/.*id=\"\([^\"]*\)\".*/\1/g;'`"
							format_name | sed 's/&gt;/>/g;
							s/&lt;/</g;s/&quot;/"/g;s/&amp;/\&/g;'

						done
					)
					}
				;;
			esac
			get_count() {
			(
				url_encoded_tags="$1"
				shift
				downloader "${p_danbooru_url}/index.php?page=dapi&s=post&q=index&tags=${url_encoded_tags}&limit=1" "${p_temp_query}"
				cat -- "${p_temp_query}" | sed -n '/^.*<posts count="\([0-9]*\)" offset="[0-9]*">.*$/{s//\1/;p;}'
				rm -- "${p_temp_query}"
			)
			}
			format_name() {
			(
				post_ext="`printf "%s\n" "${post_url}" | 
				sed 's/.*\(\.[^.]*\)$/\1/;'`"
				post_ext_length="`printf "%s" "${post_ext}" | 
				wc -c | awk '{print $1}'`"
				name_length="`expr 250 - "${post_ext_length}"`"

				# no "" !
				sed_string=`printf "%s\n" "s,post:rating,${post_rating},g;
				s,post:height,${post_height},g;
				s,post:width,${post_width},g;
				s,post:tags,${post_tags},g;
				s,post:md5,${post_md5},g;
				s,post:id,${post_id},g;" | sed 's/&/\\\&/g;'`

				name="`printf "%s\n" "${s_file_name_format}" | 
				sed -- "${sed_string}" |
				dd bs="${name_length}" count=1 2>/dev/null`"
				printf "%s %s\n" "${post_url}" "${name}${post_ext}"
			)
			}
			query() {
			(
				url_encoded_tags="$1"
				shift
				page="$1"
				shift
				limit="$1"
				shift
				downloader "${p_danbooru_url}/index.php?page=dapi&s=post&q=index&tags=${url_encoded_tags}&limit=${limit}&pid=${page}" "${p_temp_query}"
				printf "\n" >> "${p_temp_query}"
				cat -- "${p_temp_query}" | parse
				rm -- "${p_temp_query}"
			)
			}
		;;
	esac
}

# ############################################################################ #


# ################################## Main Section ############################ #

case "${l_mode}" in
	"search")
		process() {
		(
			tag_group="$1"
			shift
			url_encoded_tags="`tags_urlencode "${tag_group}"`"
			query_type="simple"
			log "message" "\t%9s %10s %s\n" "Type" "Count" "Name"
			if printf "%s\n" "${tag_group}" | grep '[: ]\{1,\}' >/dev/null; then
				query_type="deep"
			fi
			query "${query_type}" "${url_encoded_tags}" "${l_search_order}" "${l_search_limit}" | 
			while read line; do
				set -- ${line}
				type="$1"
				[ ! -z "$2" ] && shift
				count="$1"
				[ ! -z "$2" ] && shift
				tag_group="$@"
				if [ ! -z "${tag_group}" ]; then
					log "message" "\t%9s %10d %s\n" "${type}" "${count}" "${tag_group}"
				fi
			done
		)
		}
	;;
	"download")
		
		case "${l_download_mode}" in
			"onedir")
				process_files() {
				(
					file_url="$1"
					shift
					file_name="$1"
					shift
					post_number="$1"
					shift
					post_count="$1"
					shift
					post_count_length="`printf "%s" "${post_count}" | wc -c | awk '{print $1}'`"					
					log "message" "    Downloading file %s (%0${post_count_length}d/%d)..." "${file_name}" "${post_number}" "${post_count}"
					download_dir_name="`printf "%s\n" "${tag_group}" | sed 's/ /-/g;s|/||g;s|\\\||g;'`"
					mkdir -p -- "${download_dir_name}"
					file_path="${p_working_directory}/${download_dir_name}/${file_name}"
					if [ -f "${file_path}" ]; then
						printf "%s\n" "skip"
						return 0
					fi
					downloader "${file_url}" "${p_temp_image}"
					mv -- "${p_temp_image}" "${file_path}"
					log "message" "%s\n" "done"
				)
				}
			;;
			"export")
				process_files() {
				(
					file_url="$1"
					shift
					file_name="$1"
					shift
					post_number="$1"
					shift
					post_count="$1"
					shift
					log "export" "%s %d %d %s\n" "${file_url}" "${post_number}" "${post_count}" "${file_name}"
				)
				}
			;;
		esac
		process() {
		(
			tag_group="$1"
			shift
			url_encoded_tags="`tags_urlencode "${tag_group}"`"
			post_count="`get_count "${url_encoded_tags}"`"
			page_count="`expr "${post_count}" / "${l_download_limit}"`"
			if [ 0 -ne "`expr "${post_count}" % "${l_download_limit}"`" ]; then
				page_count="`expr "${page_count}" + 1`"
			fi
			page="${l_page}"
			log "message" "Starting grabbing for tags '%s'\n" "${tag_group}"
			while [ "${page}" -le "${page_count}" ]; do
				post_number="`expr '(' '(' "${page}" - 1 ')' '*' "${l_download_limit}" ')' + 1`"
				log "message" "  Switching to page %d of %d\n" "${page}" "${page_count}"
				query "${url_encoded_tags}" "${page}" "${l_download_limit}" |
				while read line; do
					set -- ${line}
					file_url="$1"
					shift
					file_name="`printf "%s\n" "$@" | sed 's|/||g;s|\\\||g;'`"
					shift	
					process_files "${file_url}" "${file_name}" "${post_number}" "${post_count}" "${post_count_length}"
					post_number="`expr "${post_number}" + 1`"
				done
				page="`expr "${page}" + 1`"
			done
			log "message" "End grabbing for tags '%s'\n" "${tag_group}"
		)
		}
	;;
esac

case "${l_engine}" in
	"danbooru")
		engine_danbooru "${l_mode}" "${l_interface}"
	;;
	"gelbooru")
		engine_gelbooru "${l_mode}" "${l_interface}"
	;;
esac

set -f
IFS=","
for tag_group in ${s_tag_list}; do
	IFS=" "
	process "${tag_group}"
done
