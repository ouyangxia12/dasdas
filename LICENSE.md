<?PHP /* $Id$ */
# This file is part of Chatty :)
# Copyright (C) 2003, 2004, 2005 Marco Olivo
#
# Chatty :) is free software; you can redistribute it and/or
# modify it under the terms of the GNU General Public License
# as published by the Free Software Foundation; either version 2
# of the License, or (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 59 Temple Place - Suite 330, Boston, MA  02111-1307, USA.

include("config.inc.php");
include("connect.inc.php");
include("utils.inc.php");
include("lang.inc.php");

/* html code for the chat */
function chat()
{
	global $username, $password;
	global $chat_title;
	$password = md5($password);
	$username = urlencode($username);

	echo "<!DOCTYPE html PUBLIC \"-//W3C//DTD HTML 4.0 Transitional//EN\">\r\n";

	echo "<HTML>\r\n";
	echo "<HEAD>\r\n";
	echo "<LINK REL=\"stylesheet\" HREF=\"style.css.php\" TYPE=\"text/css\">\r\n";
	echo "<META HTTP-EQUIV=\"Content-Type\" CONTENT=\"text/html; CHARSET=iso-8859-1\">\r\n";
	echo "<TITLE>$chat_title Chat!</TITLE>\r\n";
	echo "</HEAD>\r\n\r\n";

	echo "<FRAMESET ROWS=\"*,80\" COLS=\"*,150\" FRAMESPACING=\"0\" FRAMEBORDER=\"0\" BORDER=\"0\">\r\n";
	echo "<FRAME NAME=\"talk\" NORESIZE SRC=\"talk.php?username=$username&password=$password\" MARGINWIDTH=\"12\" MARGINHEIGHT=\"12\">\r\n";
	echo "<FRAME NAME=\"users\" SCROLLING=\"no\" NORESIZE SRC=\"users.php?username=$username&password=$password\" MARGINWIDTH=\"12\" MARGINHEIGHT=\"12\">\r\n";
	/* pick a random color for the current user */
	$color = random(9);
	echo "<FRAME NAME=\"your_msg\" SCROLLING=\"no\" NORESIZE SRC=\"sendmsg.php?color=$color&username=$username&password=$password\" MARGINWIDTH=\"12\" MARGINHEIGHT=\"12\">\r\n";
	echo "<FRAME SRC=\"logo.php\" MARGINWIDTH=\"0\" MARGINHEIGHT=\"0\" FRAMEBORDER=\"0\" SCROLLING=\"no\">\r\n";

	echo "<NOFRAMES>\r\n";

	echo "<BODY>\r\n";
	echo "Your browser doesn't support frames. Sorry :-(\r\n";
	echo "</BODY>\r\n";

	echo "</NOFRAMES>\r\n";
	echo "</FRAMESET>\r\n";

	echo "</HTML>\r\n";
}


header("Expires: Sun, 28 Dec 1997 09:32:45 GMT");
header("Cache-Control: no-cache, must-revalidate");
header("Pragma: no-cache");

/* is this the first enter on this page? */
$first_enter = !isset($username) && !isset($password);
$failed_data = false;

/* routine-checks */
if (!isset($username)) {
	$username = "";
}
if (!isset($password)) {
	$password = "";
}
if (!isset($email)) {
	$email = "";
}

/* security checks */
$username = htmlentities(urldecode(mysql_real_escape_string($username)), ENT_QUOTES);
$password = mysql_real_escape_string(stripslashes($password));

/* ok, let's see if the user can log in */
if (!$first_enter && !$failed_data) {
	if ($username == "") {
		$first_enter = true;
	}
	else if ($password == "") {
		$failed_data = true;
	}
	else if (isset($enter)) {	/* user seems to be registered: mmhh, is that true? */
		$query = "SELECT * FROM users WHERE username = '$username' AND password = '" . md5("$password") . "'";
		$result = do_the_query($chat_db, $query);

		if (mysql_num_rows($result) != 0) {	/* login ok! */
			$failed_data = false;
			/* register the successful login */
			$query = "UPDATE users SET active = 'y', sent_on = DATE_ADD(NOW(), INTERVAL $diff_timezone HOUR) WHERE username = '$username'";
			do_the_query($chat_db, $query);
			/* we warn the other users of the chat */
			$query = "INSERT INTO msg(username, msg, color, sent_on) VALUES ('', '$username', '', DATE_ADD(NOW(), INTERVAL $diff_timezone HOUR))";
			do_the_query($chat_db, $query);
		}
		else {
			$failed_data = true;
		}

		mysql_free_result($result);
	}
	else if (isset($subscribe)) {
		$query = "SELECT * FROM users WHERE username = '$username'";
		$result = do_the_query($chat_db, $query);

		if (mysql_num_rows($result) != 0 || $email == "") {	/* wrong registration: the email is empty or there is another user registered with that nickname */
			$failed_data = true;
			mysql_free_result($result);
		}
		else {
			$failed_data = false;
			/* register the new user */
			$query = "REPLACE INTO users (username, password, email, active, sent_on) VALUES ('$username', '" . md5("$password") . "', '$email', 'y', DATE_ADD(NOW(), INTERVAL $diff_timezone HOUR))";
			do_the_query($chat_db, $query);

			/* we warn the other users of the chat */
			$query = "INSERT INTO msg(username, msg, color, sent_on) VALUES ('', '$username', '', DATE_ADD(NOW(), INTERVAL $diff_timezone HOUR))";
			do_the_query($chat_db, $query);

			/* send a welcome message */
			$msg = "Dear user,\r\nwelcome in our chat.\r\nHere is the data you entered:\r\n\r\nusername: " . stripslashes($username) . "\r\npassword: $password\r\n\r\nI hope to see you soon in our chat.\r\nBye!\r\n\r\n";

			mail($email, "Welcome!", $msg, "From: $mail_address\r\n");

			mysql_free_result($result);
		}
	}
}

if (!$first_enter && !$failed_data) {	/* everything is ok: let's enter the chat! */
	chat();
	exit;
}

echo "<!DOCTYPE html PUBLIC \"-//W3C//DTD HTML 4.0 Transitional//EN\">\r\n";

echo "<HTML>\r\n";
echo "<HEAD>\r\n";
echo "<LINK REL=\"stylesheet\" HREF=\"style.css.php\" TYPE=\"text/css\">\r\n";
echo "<META HTTP-EQUIV=\"Content-Type\" CONTENT=\"text/html; CHARSET=iso-8859-1\">\r\n";
echo "<TITLE>$lang[register_title]</TITLE>\r\n";
echo "</HEAD>\r\n\r\n";

echo "<BODY>\r\n";

echo "\r\n";
echo "<!--begin login table-->\r\n";
echo "<DIV CLASS=\"center\"><TABLE BORDER=\"0\" BGCOLOR=\"$index_table_bgcolor\" CELLSPACING=\"0\" CELLPADDING=\"10\" WIDTH=\"65%\">\r\n";
echo "<TR>\r\n";
echo "<TD COLSPAN=\"2\" VALIGN=\"top\">\r\n";

echo "<DIV CLASS=\"center\"><TABLE BORDER=\"0\" BGCOLOR=\"$index_header_bgcolor\" WIDTH=\"95%\" CELLSPACING=\"0\" CELLPADDING=\"3\">\r\n";
echo "<TD WIDTH=\"50%\" VALIGN=\"top\" CLASS=\"center\">\r\n";
echo "<SPAN CLASS=\"bigwhite\">$lang[new_user]</SPAN>\r\n";
echo "</TD>\r\n";
echo "<TD WIDTH=\"50%\" VALIGN=\"top\" CLASS=\"center\">\r\n";
echo "<SPAN CLASS=\"bigwhite\">$lang[existing_user]</SPAN>\r\n";
echo "</TD>\r\n";
echo "</TR>\r\n";
echo "</TABLE></DIV>\r\n";

echo "</TD>\r\n";
echo "</TR>\r\n";

echo "<TR>\r\n";
echo "<TD WIDTH=\"50%\" VALIGN=\"top\">\r\n";
if ($failed_data && isset($subscribe)) {
	if ($email == "") {
		echo "<SPAN CLASS=\"highlight\"><B>$lang[email_reason]</B></SPAN><BR>&nbsp;<BR>\r\n";
	}
	else if ($password == "") {
		echo "<SPAN CLASS=\"highlight\"><B>$lang[pass_reason]</B></SPAN><BR>&nbsp;<BR>\r\n";
	}
	else /*if ($username == "")*/ {
		echo "<SPAN CLASS=\"highlight\"><B>$lang[dup_user]</B><BR>Thanks.</SPAN><BR>&nbsp;<BR>\r\n";
	}
}
else {
	echo "$lang[new_user_instructions] <BR>&nbsp;<BR>\r\n";
}
echo "</TD>\r\n";
echo "<TD WIDTH=\"50%\" VALIGN=\"top\">\r\n";
if ($failed_data && isset($enter)) {
	echo "<SPAN CLASS=\"highlight\"><B>$lang[wrong_login]</B></SPAN><BR>&nbsp;<BR>\r\n";
}
else {
	echo "$lang[existing_user_instructions]<BR>&nbsp;<BR>\r\n";
}
echo "</TD>\r\n";
echo "</TR>\r\n";

echo "<TR>\r\n";
echo "<TD WIDTH=\"50%\" VALIGN=\"top\">\r\n";
echo "<FORM METHOD=\"post\" ACTION=\"$PHP_SELF\">\r\n";
echo "<SPAN CLASS=\"highlight\">$lang[new_username]</SPAN><BR>\r\n";
echo "<INPUT TYPE=\"text\" NAME=\"username\" SIZE=\"15\" VALUE=\"$username\"><BR>&nbsp;<BR>\r\n";
echo "<SPAN CLASS=\"highlight\">$lang[new_password]:</SPAN><BR>\r\n";
echo "<INPUT TYPE=\"password\" NAME=\"password\" SIZE=\"15\" VALUE=\"$password\"><BR>&nbsp;<BR>\r\n";
echo "<SPAN CLASS=\"highlight\">$lang[new_email]</SPAN><BR>\r\n";
echo "<INPUT TYPE=\"text\" NAME=\"email\" SIZE=\"15\" VALUE=\"$email\"><BR>\r\n";
echo "$lang[email_disclaimer]<BR>&nbsp;<BR>\r\n";
echo "<INPUT TYPE=\"submit\" NAME=\"subscribe\" VALUE=\"$lang[subscribe]\">\r\n";
echo "</FORM>\r\n";
echo "</TD>\r\n";

echo "<TD WIDTH=\"50%\" VALIGN=\"top\">\r\n";
echo "<FORM METHOD=\"post\" ACTION=\"$PHP_SELF\">\r\n";
echo "<SPAN CLASS=\"highlight\">$lang[enter_username]</SPAN><BR>\r\n";
echo "<INPUT TYPE=\"text\" NAME=\"username\" SIZE=\"15\" VALUE=\"$username\"><BR>&nbsp;<BR>\r\n";
echo "<SPAN CLASS=\"highlight\">$lang[enter_password]</SPAN><BR>\r\n";
echo "<INPUT TYPE=\"password\" NAME=\"password\" SIZE=\"15\" VALUE=\"$password\">\r\n";
echo "<BR>&nbsp;<BR>\r\n";
echo "<INPUT TYPE=\"submit\" NAME=\"enter\" VALUE=\"$lang[enter_chat]\">\r\n";
echo "</FORM>\r\n";
echo "</TD>\r\n";
echo "</TR>\r\n";

echo "<TR>\r\n";
echo "<TD COLSPAN=\"2\" VALIGN=\"top\">\r\n";
echo "$lang[email_comments]<A HREF=\"mailto:$mail_address\">$mail_address</A>\r\n";
echo "</TD>\r\n";
echo "</TR>\r\n";

echo "<TR>\r\n";
echo "<TD COLSPAN=\"2\" VALIGN=\"top\">\r\n";
echo "<A href=\"http://www.olivo.net/software/chatty/\">Chatty :)</A> is a GPL program developed by <A href=\"http://www.olivo.net/\">Marco Olivo</A>.\r\n";
echo "</TD>\r\n";
echo "</TR>\r\n";

echo "</TABLE></DIV>\r\n";
echo "<!--end login table-->\r\n";
echo "\r\n";

echo "</BODY>\r\n";
echo "</HTML>\r\n";
?>
