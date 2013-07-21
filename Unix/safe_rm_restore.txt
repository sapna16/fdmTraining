#!/bin/bash 
#script will move files to the recycle bin and add their details to a log
#check if the user has specified a bin location
if [ ! -z $RMCFG ]; then
	RECYCLE_BIN=$RMCFG
	
elif [ -e $HOME/.rm.cfg ]; then
	eval RECYCLE_BIN=`cat $HOME/.rm.cfg`
		
else
	#set the recycle bin
	RECYCLE_BIN=$HOME/deleted
fi
#checks if there is a recycle bin; if not, makes one
if [ ! -d $RECYCLE_BIN ]; then
	mkdir $RECYCLE_BIN 2>/dev/null
fi

#checks if the filename is valid
if [ $# -eq 0 ]; then
	echo "There must be a filename"
	exit 1
fi

#if the .log does not exist, make it
if [ ! -f $RECYCLE_BIN/.restore.info ]; then
	touch $RECYCLE_BIN/.restore.info
	LOG="$RECYCLE_BIN/.restore.info"
fi


#set the flags to 0
option_i=0
option_v=0
option_r=0
while getopts :ivrR opt
do
	case $opt in
		i) 
			option_i=1 ;;
		v)
			option_v=1 ;;
		r|R)
			option_r=1 ;;
		\?) 
			echo "invalid option"
			exit 1 ;;
		esac
done
shift $(($OPTIND - 1))


function isReal(){
	local FILE=$1
	#is this a directory? Unless the -r|R flag was used, exit
	local INTEROGATIVE_IS_EMPT=$(ls $FILE|wc -l) 2>/dev/null
	if [[ -d $FILE && $option_r -eq 0 && $INTEROGATIVE_IS_EMPT -gt 0 ]]; then
		echo "File $FILE is a non-empty directory." 
		echo "Please use the -r option to remove this directory."
		exit 1
	fi
	#does this file exist?	
	if [ ! -e $FILE ];
	then
		echo "File $FILE does not exist"
		exit 1
	fi
}

function main(){

#rename the file with the inode as a prefix
	local FILE=$(/bin/readlink -f $1)
	local PAT=`/usr/bin/dirname $FILE`
	if [ -d $FILE ]; then
		local INODE=$(ls -id $FILE|cut -d" " -f1)
		local NEWNAME=$(/usr/bin/basename $FILE"_"$INODE)
	else
		local NEWNAME=$(/usr/bin/basename $FILE)"_"$(/bin/ls -i1 $FILE 2>/dev/null | /usr/bin/cut -d" " -f1 ) #2>/dev/null
	fi

	#local PAT=`/usr/bin/dirname $FILE`
	#find the inode of the parent
	local dirPATH=$(/usr/bin/dirname $PAT)
	local basePATH=$(/usr/bin/basename $PAT)
	local INODE_PARENT=$(/bin/ls -i1 $dirPATH|/bin/grep "$basePATH"|/usr/bin/cut -d" " -f1)

#if this is interactive mode, the script promts the user for y/n depending of file type
	if [ $option_i -eq 1 ]; then
		if [ -f $FILE ]; then
			if [ -s $FILE ]; then
				read -p "Do you want to remove $FILE , a regular file of nonzero size? y/n " ans
			else
				read -p "Do you want to remove $FILE , a regular empty file? y/n " ans
			fi
		fi
		if [ -d $FILE ]; then
			if [ -s $FILE ]; then
				read -p "Do you want to remove $FILE , a directory of nonzero size? y/n " ans
			else
				read -p "Do you want to remove $FILE , an empty directory? y/n " ans
			fi
		fi
		if [ $ans = n ]; then
			continue
		fi
	fi
	local CHECK=$(/bin/ls -a $FILE|/usr/bin/wc -l) 2>/dev/null
	if [[ -d $FILE && $CHECK -gt 0 ]]
	then
		local N=$(/bin/ls $FILE) 2>/dev/null
		for f in $N
		do
			if [ $option_i -eq 1 ]; then
				read -p "Do you wish to descend into directory $FILE? y/n" INTER_DOWN
				if [ $INTER_DOWN != "y" ]; then
					continue
				else
					main $FILE"/"$f
				fi
			else
				main $FILE"/"$f
			fi
		done
	fi
	#move the file	
	/bin/mv $FILE $RECYCLE_BIN/$NEWNAME 2>/dev/null
	#add the file details to the log
	local NAME=$(/bin/readlink -f $FILE)
	echo $NEWNAME":"$NAME":"$PAT"_"$INODE_PARENT >> $RECYCLE_BIN/.restore.info
	#if this is verbose mode, the script echoes its actions
	if [ $option_v -eq 1 ]; then
		echo "Removing file $FILE "
	fi
}

for file in $*
do
	isReal $file
	main $file
done

