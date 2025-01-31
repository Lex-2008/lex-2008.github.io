title=Conditional formatting in Python
uuid=3099ed35-85f1-4586-b567-ee72b4a4b5db
PROCESSOR=Markdown.pl
intro=Greately inspired by $if(...) conditions in Winamp Advanced Title Formatting, I wanted to add something like this to Python.
tags=python
created=2014-02-01
modified=2016-06-16
modified_now=1


Script uses quite a lot of regexps, but looks right. Nested conditions are not supported, though. Also note that about half of the code is comments :-)

	from collections import defaultdict
	from string import Formatter
	import re
	
	# converts a dict to defaultdict:
	# where missing values are also nesteddefaultdict:
	#   a = nesteddefaultdict()
	#   a['b']['c'] = 'd'
	#   a == {'b':{'c':'d'}}
	# but that's true only on top level (it's a 'shallow copy'):
	#   a = nesteddefaultdict({'b': {'c':'d'}})
	#   print a['b']['d'] <= fails :(
	#
	# replaceable with an empty string:
	#   str(a) = ''
	# useful when:
	#   a = nesteddefaultdict({'foo': 1})
	#   print a['bar'] => prints '' instead of failing
	def nesteddefaultdict(basedict={}):
		class d2(defaultdict):
			def __str__(self):
				return ''
		result = d2(nesteddefaultdict)
		result.update(basedict)
		return result
	
	# format a string, almost like 'str'.format() does.
	# Args:
	#   text - format string
	#   data - dict, defaultdict or nesteddefaultdict
	#
	# in following examples we assume that:
	#   data=nesteddefaultdict({'foo':1})
	# uses {keyword} expansions, like this:
	#   format('foo equals {foo}', data) => 'foo equals 1'
	# 'safe' substitution is a feature of defaultdict:
	#   format('bar equals {bar}', data) => 'bar equals '
	# same for [indexes]:
	#   format('a is {a[foo]} and {a[bar]}', {'a':data}) => 'a is 1 and '
	# ===Conditional formatting===
	# {{?...?}} -- shows content only if it has non-empty {} expansions:
	#   format('{{? foo={foo} ?}}', data) => ' foo=1 '
	#   format('{{? bar={bar} ?}}', data) => ''
	# {{?...?...|...?}} -- if-then-else ternary operator:
	#   shows second group if after substitution first group has non-space character, third group otherwise:
	#   format('a has: {{? {a[foo]} ? foo={a[foo]} | no foo ?}}', {'a':data}) => 'a has:  foo=1 '
	#   format('a has: {{? {a[bar]} ? bar={a[bar]} | no bar ?}}', {'a':data}) => 'a has:  no bar '
	# {{?...==...?...|...?}} -- if-then-else ternary operator with comparison
	# {{?...!=...?...|...?}} -- if-then-else ternary operator with comparison
	#   format('{{? {foo}==1 ? foo equals one | different ?}}', data) => ' foo equals one '
	#   format('{{? {bar}!= ? bar is non-empty | bar is empty ?}}', data) => ' bar is empty '
	def format(text, data):
		dict_with_nothing=nesteddefaultdict()
		# {{?a?}}
		text = re.split('{{\?([^?|]*)\?}}', text)
		# delete those groups where data has no visible effect
		for i in range(1, len(text), 2):
			group_with_data = Formatter().vformat(text[i],[],data)
			group_with_nothing = Formatter().vformat(text[i],[],dict_with_nothing)
			if group_with_data == group_with_nothing:
				text[i] = ''
		text = ''.join(text)
		# apply all format options
		text = Formatter().vformat(text,[],data)
		# {{?a[!=]b?c|d?}}
		text = re.split('{\?([^?=]*)([=!])=([^?]*)\?([^|?]*)\|([^?]*)\?}', text)
		# select one of two groups (i+3 or i+4) based on (i+1) comparison of others (i and i+2)
		for i in range(1, len(text), 6):
			if (text[i].strip()==text[i+2].strip()) == (text[i+1]=='='):
				text[i+4] = ''
			else:
				text[i+3] = ''
			text[i] = text[i+1] = text[i+2] = ''
		text = ''.join(text)
		# {{?a?b|c?}}
		text = re.split('{\?([^?]*)\?([^|?]*)\|([^?]*)\?}', text)
		# select one of two groups (i+1 or i+2) based on third one (i)
		for i in range(1, len(text), 4):
			if re.search('\S', text[i]):
				text[i+2] = ''
			else:
				text[i+1] = ''
			text[i] = ''
		text = ''.join(text)
		return text
	
	if __name__ == "__main__":
		a = nesteddefaultdict()
		a['b']['c'] = 'd'
		print a == {'b':{'c':'d'}}

		data = nesteddefaultdict({'foo':1})
		print format('"foo equals {foo}"', data), '"foo equals 1"'
		print format('"bar equals {bar}"', data), '"bar equals "'
		print format('"a is {a[foo]} and {a[bar]}"', {'a':data}), '"a is 1 and "'
		print format('"{{? foo={foo} ?}}"', data), '" foo=1 "'
		print format('"{{? bar={bar} ?}}"', data), '""'
		print format('"a has: {{? {a[foo]} ? foo={a[foo]} | no foo ?}}"', {'a':data}), '"a has:  foo=1 "'
		print format('"a has: {{? {a[bar]} ? bar={a[bar]} | no bar ?}}"', {'a':data}), '"a has:  no bar "'
		print format('"{{? {foo}==1 ? foo equals one | different ?}}"', data), '" foo equals one "'
		print format('"{{? {bar}!= ? bar is non-empty | bar is empty ?}}"', data), '" bar is empty "'

<script src="/microlight.js"></script>
