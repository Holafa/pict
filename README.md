/**
 * License: Apache 2.0 with LLVM Exception or GPL v3
 *
 * Author: Jesse Laning
 */

#ifndef ARGPARSE_H
#define ARGPARSE_H

#include <algorithm>
#include <cctype>
#include <cstring>
#include <iomanip>
#include <iostream>
#include <locale>
#include <map>
#include <numeric>
#include <sstream>
#include <stdexcept>
#include <string>
#include <unordered_map>
#include <vector>

namespace argparse {
	namespace detail {
		static inline bool _not_space(int ch) { return !std::isspace(ch); }
		static inline void _ltrim(std::string &s, bool(*f)(int) = _not_space) {
			s.erase(s.begin(), std::find_if(s.begin(), s.end(), f));
		}
		static inline void _rtrim(std::string &s, bool(*f)(int) = _not_space) {
			s.erase(std::find_if(s.rbegin(), s.rend(), f).base(), s.end());
		}
		static inline void _trim(std::string &s, bool(*f)(int) = _not_space) {
			_ltrim(s, f);
			_rtrim(s, f);
		}
		static inline std::string _ltrim_copy(std::string s,
			bool(*f)(int) = _not_space) {
			_ltrim(s, f);
			return s;
		}
		static inline std::string _rtrim_copy(std::string s,
			bool(*f)(int) = _not_space) {
			_rtrim(s, f);
			return s;
		}
		static inline std::string _trim_copy(std::string s,
			bool(*f)(int) = _not_space) {
			_trim(s, f);
			return s;
		}
		template <typename InputIt>
		static inline std::string _join(InputIt begin, InputIt end,
			const std::string &separator = " ") {
			std::ostringstream ss;
			if (begin != end) {
				ss << *begin++;
			}
			while (begin != end) {
				ss << separator;
				ss << *begin++;
			}
			return ss.str();
		}
		static inline bool _is_number(const std::string &arg) {
			std::istringstream iss(arg);
			float f;
			iss >> std::noskipws >> f;
			return iss.eof() && !iss.fail();
		}

		static inline int _find_equal(const std::string &s) {
			for (size_t i = 0; i < s.length(); ++i) {
				// if find graph symbol before equal, end search
				// i.e. don't accept --asd)f=0 arguments
				// but allow --asd_f and --asd-f arguments
				if (std::ispunct(static_cast<int>(s[i]))) {
					if (s[i] == '=') {
						return static_cast<int>(i);
					}
					else if (s[i] == '_' || s[i] == '-') {
						continue;
					}
					return -1;
				}
			}
			return -1;
		}

		static inline size_t _find_name_end(const std::string &s) {
			size_t i;
			for (i = 0; i < s.length(); ++i) {
				if (std::ispunct(static_cast<int>(s[i]))) {
					break;
				}
			}
			return i;
		}

		namespace is_vector_impl {
			template <typename T>
			struct is_vector : std::false_type {};
			template <typename... Args>
			struct is_vector<std::vector<Args...>> : std::true_type {};
		}  // namespace is_vector_impl

		// type trait to utilize the implementation type traits as well as decay the
		// type
		template <typename T>
		struct is_vector {
			static constexpr bool const value =
				is_vector_impl::is_vector<typename std::decay<T>::type>::value;
		};
	}  // namespace detail

	class ArgumentParser {
	private:
	public:
		class Argument;

		class Result {
		public:
			Result() {}
			Result(std::string err) noexcept : _error(true), _what(err) {}

			operator bool() const { return _error; }

			friend std::ostream &operator<<(std::ostream &os, const Result &dt);

			const std::string &what() const { return _what; }

		private:
			bool _error{ false };
			std::string _what{};
		};

		class Argument {
		public:
			enum Position : int { LAST = -1, DONT_CARE = -2 };
			enum Count : int { ANY = -1 };

			Argument &name(const std::string &name) {
				_names.push_back(name);
				return *this;
			}

			Argument &names(std::vector<std::string> names) {
				_names.insert(_names.end(), names.begin(), names.end());
				return *this;
			}

			Argument &description(const std::string &description) {
				_desc = description;
				return *this;
			}

			Argument &required(bool req) {
				_required = req;
				return *this;
			}

			Argument &position(int position) {
				if (position != Position::LAST) {
					// position + 1 because technically argument zero is the name of the
					// executable
					_position = position + 1;
				}
				else {
					_position = position;
				}
				return *this;
			}

			Argument &count(int count) {
				_count = count;
				return *this;
			}

			bool found() const { return _found; }

			template <typename T>
			typename std::enable_if<detail::is_vector<T>::value, T>::type get() {
				T t = T();
				typename T::value_type vt;
				for (auto &s : _values) {
					std::istringstream in(s);
					in >> vt;
					t.push_back(vt);
				}
				return t;
			}

			template <typename T>
			typename std::enable_if<!detail::is_vector<T>::value, T>::type get() {
				std::istringstream in(get<std::string>());
				T t = T();
				in >> t >> std::ws;
				return t;
			}

		private:
			Argument(const std::string &name, const std::string &desc,
				bool required = false)
				: _desc(desc), _required(required) {
				_names.push_back(name);
			}

			Argument() {}

			friend class ArgumentParser;
			int _position{ Position::DONT_CARE };
			int _count{ Count::ANY };
			std::vector<std::string> _names{};
			std::string _desc{};
			bool _found{ false };
			bool _required{ false };
			int _index{ -1 };

			std::vector<std::string> _values{};
		};

		ArgumentParser(const std::string &bin, const std::string &desc)
			: _bin(bin), _desc(desc) {}

		Argument &add_argument() {
			_arguments.push_back({});
			_arguments.back()._index = static_cast<int>(_arguments.size()) - 1;
			return _arguments.back();
		}

		Argument &add_argument(const std::string &name, const std::string &long_name,
			const std::string &desc, const bool required = false) {
			_arguments.push_back(Argument(name, desc, required));
			_arguments.back()._names.push_back(long_name);
			_arguments.back()._index = static_cast<int>(_arguments.size()) - 1;
			return _arguments.back();
		}

		Argument &add_argument(const std::string &name, const std::string &desc,
			const bool required = false) {
			_arguments.push_back(Argument(name, desc, required));
			_arguments.back()._index = static_cast<int>(_arguments.size()) - 1;
			return _arguments.back();
		}

		void print_help(size_t count = 0, size_t page = 0) {
			if (page * count > _arguments.size()) {
				return;
			}
			if (page == 0) {
				std::cout << "Usage: " << _bin;
				if (_positional_arguments.empty()) {
					std::cout << " [options...]" << std::endl;
				}
				else {
					int current = 1;
					for (auto &v : _positional_arguments) {
						if (v.first != Argument::Position::LAST) {
							for (; current < v.first; current++) {
								std::cout << " [" << current << "]";
							}
							std::cout
								<< " ["
								<< detail::_ltrim_copy(
									_arguments[static_cast<size_t>(v.second)]._names[0],
									[](int c) -> bool { return c != static_cast<int>('-'); })
								<< "]";
						}
					}
					auto it = _positional_arguments.find(Argument::Position::LAST);
					if (it == _positional_arguments.end()) {
						std::cout << " [options...]";
					}
					else {
						std::cout
							<< " [options...] ["
							<< detail::_ltrim_copy(
								_arguments[static_cast<size_t>(it->second)]._names[0],
								[](int c) -> bool { return c != static_cast<int>('-'); })
							<< "]";
					}
					std::cout << std::endl;
				}
				std::cout << "Options:" << std::endl;
			}
			if (count == 0) {
				page = 0;
				count = _arguments.size();
			}
			for (size_t i = page * count;
				i < std::min<size_t>(page * count + count, _arguments.size()); i++) {
				Argument &a = _arguments[i];
				std::string name = a._names[0];
				for (size_t n = 1; n < a._names.size(); ++n) {
					name.append(", " + a._names[n]);
				}
				std::cout << "    " << std::setw(23) << std::left << name << std::setw(23)
					<< a._desc;
				if (a._required) {
					std::cout << " (Required)";
				}
				std::cout << std::endl;
			}
		}

		Result parse(int argc, const char *argv[]) {
			Result err;
			if (argc > 1) {
				// build name map
				for (auto &a : _arguments) {
					for (auto &n : a._names) {
						std::string name = detail::_ltrim_copy(
							n, [](int c) -> bool { return c != static_cast<int>('-'); });
						if (_name_map.find(name) != _name_map.end()) {
							return Result("Duplicate of argument name: " + n);
						}
						_name_map[name] = a._index;
					}
					if (a._position >= 0 || a._position == Argument::Position::LAST) {
						_positional_arguments[a._position] = a._index;
					}
				}
				if (err) {
					return err;
				}

				// parse
				std::string current_arg;
				size_t arg_len;
				for (int argv_index = 1; argv_index < argc; ++argv_index) {
					current_arg = std::string(argv[argv_index]);
					arg_len = current_arg.length();
					if (arg_len == 0) {
						continue;
					}
					if (_help_enabled && (current_arg == "-h" || current_arg == "--help")) {
						_arguments[static_cast<size_t>(_name_map["help"])]._found = true;
					}
					else if (argv_index == argc - 1 &&
						_positional_arguments.find(Argument::Position::LAST) !=
						_positional_arguments.end()) {
						err = _end_argument();
						Result b = err;
						err = _add_value(current_arg, Argument::Position::LAST);
						if (b) {
							return b;
						}
						if (err) {
							return err;
						}
					}
					else if (arg_len >= 2 &&
						!detail::_is_number(current_arg)) {  // ignores the case if
															 // the arg is just a -
			   // look for -a (short) or --arg (long) args
						if (current_arg[0] == '-') {
							err = _end_argument();
							if (err) {
								return err;
							}
							// look for --arg (long) args
							if (current_arg[1] == '-') {
								err = _begin_argument(current_arg.substr(2), true, argv_index);
								if (err) {
									return err;
								}
							}
							else {  // short args
								err = _begin_argument(current_arg.substr(1), false, argv_index);
								if (err) {
									return err;
								}
							}
						}
						else {  // argument value
							err = _add_value(current_arg, argv_index);
							if (err) {
								return err;
							}
						}
					}
					else {  // argument value
						err = _add_value(current_arg, argv_index);
						if (err) {
							return err;
						}
					}
				}
			}
			if (_help_enabled && exists("help")) {
				return Result();
			}
			err = _end_argument();
			if (err) {
				return err;
			}
			for (auto &p : _positional_arguments) {
				Argument &a = _arguments[static_cast<size_t>(p.second)];
				if (a._values.size() > 0 && a._values[0][0] == '-') {
					std::string name = detail::_ltrim_copy(a._values[0], [](int c) -> bool {
						return c != static_cast<int>('-');
					});
					if (_name_map.find(name) != _name_map.end()) {
						if (a._position == Argument::Position::LAST) {
							return Result(
								"Poisitional argument expected at the end, but argument " +
								a._values[0] + " found instead");
						}
						else {
							return Result("Poisitional argument expected in position " +
								std::to_string(a._position) + ", but argument " +
								a._values[0] + " found instead");
						}
					}
				}
			}
			for (auto &a : _arguments) {
				if (a._required && !a._found) {
					return Result("Required argument not found: " + a._names[0]);
				}
				if (a._position >= 0 && argc >= a._position && !a._found) {
					return Result("Argument " + a._names[0] + " expected in position " +
						std::to_string(a._position));
				}
			}
			return Result();
		}

		void enable_help() {
			add_argument("-h", "--help", "Shows this page", false);
			_help_enabled = true;
		}

		bool exists(const std::string &name) const {
			std::string n = detail::_ltrim_copy(
				name, [](int c) -> bool { return c != static_cast<int>('-'); });
			auto it = _name_map.find(n);
			if (it != _name_map.end()) {
				return _arguments[static_cast<size_t>(it->second)]._found;
			}
			return false;
		}

		template <typename T>
		T get(const std::string &name) {
			auto t = _name_map.find(name);
			if (t != _name_map.end()) {
				return _arguments[static_cast<size_t>(t->second)].get<T>();
			}
			return T();
		}

	private:
		Result _begin_argument(const std::string &arg, bool longarg, int position) {
			auto it = _positional_arguments.find(position);
			if (it != _positional_arguments.end()) {
				Result err = _end_argument();
				Argument &a = _arguments[static_cast<size_t>(it->second)];
				a._values.push_back((longarg ? "--" : "-") + arg);
				a._found = true;
				return err;
			}
			if (_current != -1) {
				return Result("Current argument left open");
			}
			size_t name_end = detail::_find_name_end(arg);
			std::string arg_name = arg.substr(0, name_end);
			if (longarg) {
				int equal_pos = detail::_find_equal(arg);
				auto nmf = _name_map.find(arg_name);
				if (nmf == _name_map.end()) {
					return Result("Unrecognized command line option '" + arg_name + "'");
				}
				_current = nmf->second;
				_arguments[static_cast<size_t>(nmf->second)]._found = true;
				if (equal_pos == 0 ||
					(equal_pos < 0 &&
						arg_name.length() < arg.length())) {  // malformed argument
					return Result("Malformed argument: " + arg);
				}
				else if (equal_pos > 0) {
					std::string arg_value = arg.substr(name_end + 1);
					_add_value(arg_value, position);
				}
			}
			else {
				Result r;
				if (arg_name.length() == 1) {
					return _begin_argument(arg, true, position);
				}
				else {
					for (char &c : arg_name) {
						r = _begin_argument(std::string(1, c), true, position);
						if (r) {
							return r;
						}
						r = _end_argument();
						if (r) {
							return r;
						}
					}
				}
			}
			return Result();
		}

		Result _add_value(const std::string &value, int location) {
			if (_current >= 0) {
				Result err;
				Argument &a = _arguments[static_cast<size_t>(_current)];
				if (a._count >= 0 && static_cast<int>(a._values.size()) >= a._count) {
					err = _end_argument();
					if (err) {
						return err;
					}
					goto unnamed;
				}
				a._values.push_back(value);
				if (a._count >= 0 && static_cast<int>(a._values.size()) >= a._count) {
					err = _end_argument();
					if (err) {
						return err;
					}
				}
				return Result();
			}
			else {
			unnamed:
				auto it = _positional_arguments.find(location);
				if (it != _positional_arguments.end()) {
					Argument &a = _arguments[static_cast<size_t>(it->second)];
					a._values.push_back(value);
					a._found = true;
				}
				// TODO
				return Result();
			}
		}

		Result _end_argument() {
			if (_current >= 0) {
				Argument &a = _arguments[static_cast<size_t>(_current)];
				_current = -1;
				if (static_cast<int>(a._values.size()) < a._count) {
					return Result("Too few arguments given for " + a._names[0]);
				}
				if (a._count >= 0) {
					if (static_cast<int>(a._values.size()) > a._count) {
						return Result("Too many arguments given for " + a._names[0]);
					}
				}
			}
			return Result();
		}

		bool _help_enabled{ false };
		int _current{ -1 };
		std::string _bin{};
		std::string _desc{};
		std::vector<Argument> _arguments{};
		std::map<int, int> _positional_arguments{};
		std::map<std::string, int> _name_map{};
	};

	std::ostream &operator<<(std::ostream &os, const ArgumentParser::Result &r) {
		os << r.what();
		return os;
	}
	template <>
	inline std::string ArgumentParser::Argument::get<std::string>() {
		return detail::_join(_values.begin(), _values.end());
	}
	template <>
	inline std::vector<std::string>
		ArgumentParser::Argument::get<std::vector<std::string>>() {
		return _values;
	}

}  // namespace argparse
#endif
/*
 * This file is part of the VanitySearch distribution (https://github.com/JeanLucPons/VanitySearch).
 * Copyright (c) 2019 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#include "Base58.h"

#include <algorithm>
#include <string.h>
#include <cstdint>

/** All alphanumeric characters except for "0", "I", "O", and "l" */
static const char *pszBase58 = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";

static const int8_t b58digits_map[] = {
    -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
            -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
            -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
            -1, 0, 1, 2, 3, 4, 5, 6,  7, 8, -1, -1, -1, -1, -1, -1,
            -1, 9, 10, 11, 12, 13, 14, 15, 16, -1, 17, 18, 19, 20, 21, -1,
            22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, -1, -1, -1, -1, -1,
            -1, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, -1, 44, 45, 46,
            47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, -1, -1, -1, -1, -1,
        };

bool DecodeBase58(const char *psz, std::vector<uint8_t> &vch)
{

    uint8_t digits[256];

    // Skip and count leading '1'
    int zeroes = 0;
    while (*psz == '1') {
        zeroes++;
        psz++;
    }

    int length = (int)strlen(psz);

    // Process the characters
    int digitslen = 1;
    digits[0] = 0;
    for (int i = 0; i < length; i++) {

        // Decode base58 character
        if (psz[i] & 0x80)
            return false;

        int8_t c = b58digits_map[psz[i]];
        if (c < 0)
            return false;

        uint32_t carry = (uint32_t)c;
        for (int j = 0; j < digitslen; j++) {
            carry += (uint32_t)(digits[j]) * 58;
            digits[j] = (uint8_t)(carry % 256);
            carry /= 256;
        }
        while (carry > 0) {
            digits[digitslen++] = (uint8_t)(carry % 256);
            carry /= 256;
        }

    }

    vch.clear();
    vch.reserve(zeroes + digitslen);
    // zeros
    for (int i = 0; i < zeroes; i++)
        vch.push_back(0);

    // reverse
    for (int i = 0; i < digitslen; i++)
        vch.push_back(digits[digitslen - 1 - i]);

    return true;

}

std::string EncodeBase58(const unsigned char *pbegin, const unsigned char *pend)
{

    std::string ret;
    unsigned char digits[256];

    // Skip leading zeroes
    while (pbegin != pend && *pbegin == 0) {
        ret.push_back('1');
        pbegin++;
    }
    int length = (int)(pend - pbegin);

    int digitslen = 1;
    digits[0] = 0;
    for (int i = 0; i < length; i++) {
        uint32_t carry = pbegin[i];
        for (int j = 0; j < digitslen; j++) {
            carry += (uint32_t)(digits[j]) << 8;
            digits[j] = (unsigned char)(carry % 58);
            carry /= 58;
        }
        while (carry > 0) {
            digits[digitslen++] = (unsigned char)(carry % 58);
            carry /= 58;
        }
    }

    // reverse
    for (int i = 0; i < digitslen; i++)
        ret.push_back(pszBase58[digits[digitslen - 1 - i]]);

    return ret;

}

std::string EncodeBase58(const std::vector<unsigned char> &vch)
{
    return EncodeBase58(vch.data(), vch.data() + vch.size());
}

bool DecodeBase58(const std::string &str, std::vector<unsigned char> &vchRet)
{
    return DecodeBase58(str.c_str(), vchRet);
}
/*
 * This file is part of the VanitySearch distribution (https://github.com/JeanLucPons/VanitySearch).
 * Copyright (c) 2019 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#ifndef BASE58_H
#define BASE58_H

#include <string>
#include <vector>

/**
 * Encode a byte sequence as a base58-encoded string.
 * pbegin and pend cannot be nullptr, unless both are.
 */
std::string EncodeBase58(const unsigned char *pbegin, const unsigned char *pend);

/**
 * Encode a byte vector as a base58-encoded string
 */
std::string EncodeBase58(const std::vector<unsigned char> &vch);

/**
 * Decode a base58-encoded string (psz) into a byte vector (vchRet).
 * return true if decoding is successful.
 * psz cannot be nullptr.
 */
bool DecodeBase58(const char *psz, std::vector<unsigned char> &vchRet);

/**
 * Decode a base58-encoded string (str) into a byte vector (vchRet).
 * return true if decoding is successful.
 */
bool DecodeBase58(const std::string &str, std::vector<unsigned char> &vchRet);


#endif // BASE58_H
/* Copyright (c) 2017 Pieter Wuille
 *
 * Permission is hereby granted, free of charge, to any person obtaining a copy
 * of this software and associated documentation files (the "Software"), to deal
 * in the Software without restriction, including without limitation the rights
 * to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 * copies of the Software, and to permit persons to whom the Software is
 * furnished to do so, subject to the following conditions:
 *
 * The above copyright notice and this permission notice shall be included in
 * all copies or substantial portions of the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 * AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 * LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 * OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
 * THE SOFTWARE.
 */
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <ctype.h>

#include "Bech32.h"

uint32_t bech32_polymod_step(uint32_t pre)
{
    uint8_t b = pre >> 25;
    return ((pre & 0x1FFFFFF) << 5) ^
           (-((b >> 0) & 1) & 0x3b6a57b2UL) ^
           (-((b >> 1) & 1) & 0x26508e6dUL) ^
           (-((b >> 2) & 1) & 0x1ea119faUL) ^
           (-((b >> 3) & 1) & 0x3d4233ddUL) ^
           (-((b >> 4) & 1) & 0x2a1462b3UL);
}

static const char *charset = "qpzry9x8gf2tvdw0s3jn54khce6mua7l";

static const int8_t charset_rev[128] = {
    -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
            -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
            -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
            15, -1, 10, 17, 21, 20, 26, 30,  7,  5, -1, -1, -1, -1, -1, -1,
            -1, 29, -1, 24, 13, 25,  9,  8, 23, -1, 18, 22, 31, 27, 19, -1,
            1,  0,  3, 16, 11, 28, 12, 14,  6,  4,  2, -1, -1, -1, -1, -1,
            -1, 29, -1, 24, 13, 25,  9,  8, 23, -1, 18, 22, 31, 27, 19, -1,
            1,  0,  3, 16, 11, 28, 12, 14,  6,  4,  2, -1, -1, -1, -1, -1
        };

int bech32_encode(char *output, const char *hrp, const uint8_t *data, size_t data_len)
{
    uint32_t chk = 1;
    size_t i = 0;
    while (hrp[i] != 0) {
        int ch = hrp[i];
        if (ch < 33 || ch > 126) {
            return 0;
        }

        if (ch >= 'A' && ch <= 'Z')
            return 0;
        chk = bech32_polymod_step(chk) ^ (ch >> 5);
        ++i;
    }
    if (i + 7 + data_len > 90)
        return 0;
    chk = bech32_polymod_step(chk);
    while (*hrp != 0) {
        chk = bech32_polymod_step(chk) ^ (*hrp & 0x1f);
        *(output++) = *(hrp++);
    }
    *(output++) = '1';
    for (i = 0; i < data_len; ++i) {
        if (*data >> 5)
            return 0;
        chk = bech32_polymod_step(chk) ^ (*data);
        *(output++) = charset[*(data++)];
    }
    for (i = 0; i < 6; ++i) {
        chk = bech32_polymod_step(chk);
    }
    chk ^= 1;
    for (i = 0; i < 6; ++i) {
        *(output++) = charset[(chk >> ((5 - i) * 5)) & 0x1f];
    }
    *output = 0;
    return 1;
}

int bech32_decode_nocheck(uint8_t *data, size_t *data_len, const char *input)
{

    uint8_t acc = 0;
    uint8_t acc_len = 8;
    size_t out_len = 0;

    size_t input_len = strlen(input);
    for (int i = 0; i < input_len; i++) {

        if (input[i] & 0x80)
            return false;

        int8_t c = charset_rev[tolower(input[i])];
        if (c < 0)
            return false;

        if (acc_len >= 5) {
            acc |= c << (acc_len - 5);
            acc_len -= 5;
        } else {
            int shift = 5 - acc_len;
            data[out_len++] = acc | (c >> shift);
            acc_len = 8 - shift;
            acc = c << acc_len;
        }

    }

    data[out_len++] = acc;
    *data_len = out_len;

    return true;

}

int bech32_decode(char *hrp, uint8_t *data, size_t *data_len, const char *input)
{
    uint32_t chk = 1;
    size_t i;
    size_t input_len = strlen(input);
    size_t hrp_len;
    int have_lower = 0, have_upper = 0;
    if (input_len < 8 || input_len > 90) {
        return 0;
    }
    *data_len = 0;
    while (*data_len < input_len && input[(input_len - 1) - *data_len] != '1') {
        ++(*data_len);
    }
    hrp_len = input_len - (1 + *data_len);
    if (1 + *data_len >= input_len || *data_len < 6) {
        return 0;
    }
    *(data_len) -= 6;
    for (i = 0; i < hrp_len; ++i) {
        int ch = input[i];
        if (ch < 33 || ch > 126) {
            return 0;
        }
        if (ch >= 'a' && ch <= 'z') {
            have_lower = 1;
        } else if (ch >= 'A' && ch <= 'Z') {
            have_upper = 1;
            ch = (ch - 'A') + 'a';
        }
        hrp[i] = ch;
        chk = bech32_polymod_step(chk) ^ (ch >> 5);
    }
    hrp[i] = 0;
    chk = bech32_polymod_step(chk);
    for (i = 0; i < hrp_len; ++i) {
        chk = bech32_polymod_step(chk) ^ (input[i] & 0x1f);
    }
    ++i;
    while (i < input_len) {
        int v = (input[i] & 0x80) ? -1 : charset_rev[(int)input[i]];
        if (input[i] >= 'a' && input[i] <= 'z')
            have_lower = 1;
        if (input[i] >= 'A' && input[i] <= 'Z')
            have_upper = 1;
        if (v == -1) {
            return 0;
        }
        chk = bech32_polymod_step(chk) ^ v;
        if (i + 6 < input_len) {
            data[i - (1 + hrp_len)] = v;
        }
        ++i;
    }
    if (have_lower && have_upper) {
        return 0;
    }
    return chk == 1;
}

static int convert_bits(uint8_t *out, size_t *outlen, int outbits, const uint8_t *in, size_t inlen, int inbits, int pad)
{
    uint32_t val = 0;
    int bits = 0;
    uint32_t maxv = (((uint32_t)1) << outbits) - 1;
    while (inlen--) {
        val = (val << inbits) | *(in++);
        bits += inbits;
        while (bits >= outbits) {
            bits -= outbits;
            out[(*outlen)++] = (val >> bits) & maxv;
        }
    }
    if (pad) {
        if (bits) {
            out[(*outlen)++] = (val << (outbits - bits)) & maxv;
        }
    } else if (((val << (outbits - bits)) & maxv) || bits >= inbits) {
        return 0;
    }
    return 1;
}

int segwit_addr_encode(char *output, const char *hrp, int witver, const uint8_t *witprog, size_t witprog_len)
{
    uint8_t data[65];
    size_t datalen = 0;
    if (witver > 16)
        return 0;
    if (witver == 0 && witprog_len != 20 && witprog_len != 32)
        return 0;
    if (witprog_len < 2 || witprog_len > 40)
        return 0;
    data[0] = witver;
    convert_bits(data + 1, &datalen, 5, witprog, witprog_len, 8, 1);
    ++datalen;
    return bech32_encode(output, hrp, data, datalen);
}

int segwit_addr_decode(int *witver, uint8_t *witdata, size_t *witdata_len, const char *hrp, const char *addr)
{
    uint8_t data[84];
    char hrp_actual[84];
    size_t data_len;
    if (!bech32_decode(hrp_actual, data, &data_len, addr))
        return 0;
    if (data_len == 0 || data_len > 65)
        return 0;
    if (strncmp(hrp, hrp_actual, 84) != 0)
        return 0;
    if (data[0] > 16)
        return 0;
    *witdata_len = 0;
    if (!convert_bits(witdata, witdata_len, 8, data + 1, data_len - 1, 5, 0))
        return 0;
    if (*witdata_len < 2 || *witdata_len > 40)
        return 0;
    if (data[0] == 0 && *witdata_len != 20 && *witdata_len != 32)
        return 0;
    *witver = data[0];
    return 1;
}
/* Copyright (c) 2017 Pieter Wuille
 *
 * Permission is hereby granted, free of charge, to any person obtaining a copy
 * of this software and associated documentation files (the "Software"), to deal
 * in the Software without restriction, including without limitation the rights
 * to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 * copies of the Software, and to permit persons to whom the Software is
 * furnished to do so, subject to the following conditions:
 *
 * The above copyright notice and this permission notice shall be included in
 * all copies or substantial portions of the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 * AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 * LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 * OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
 * THE SOFTWARE.
 */

#ifndef _SEGWIT_ADDR_H_
#define _SEGWIT_ADDR_H_ 1

#include <stdint.h>

/** Encode a SegWit address
 *
 *  Out: output:   Pointer to a buffer of size 73 + strlen(hrp) that will be
 *                 updated to contain the null-terminated address.
 *  In:  hrp:      Pointer to the null-terminated human readable part to use
 *                 (chain/network specific).
 *       ver:      Version of the witness program (between 0 and 16 inclusive).
 *       prog:     Data bytes for the witness program (between 2 and 40 bytes).
 *       prog_len: Number of data bytes in prog.
 *  Returns 1 if successful.
 */
int segwit_addr_encode(
        char *output,
        const char *hrp,
        int ver,
        const uint8_t *prog,
        size_t prog_len
);

/** Decode a SegWit address
 *
 *  Out: ver:      Pointer to an int that will be updated to contain the witness
 *                 program version (between 0 and 16 inclusive).
 *       prog:     Pointer to a buffer of size 40 that will be updated to
 *                 contain the witness program bytes.
 *       prog_len: Pointer to a size_t that will be updated to contain the length
 *                 of bytes in prog.
 *       hrp:      Pointer to the null-terminated human readable part that is
 *                 expected (chain/network specific).
 *       addr:     Pointer to the null-terminated address.
 *  Returns 1 if successful.
 */
int segwit_addr_decode(
        int *ver,
        uint8_t *prog,
        size_t *prog_len,
        const char *hrp,
        const char *addr
);

/** Encode a Bech32 string
 *
 *  Out: output:  Pointer to a buffer of size strlen(hrp) + data_len + 8 that
 *                will be updated to contain the null-terminated Bech32 string.
 *  In: hrp :     Pointer to the null-terminated human readable part.
 *      data :    Pointer to an array of 5-bit values.
 *      data_len: Length of the data array.
 *  Returns 1 if successful.
 */
int bech32_encode(
        char *output,
        const char *hrp,
        const uint8_t *data,
        size_t data_len
);

/** Decode a Bech32 string
 *
 *  Out: hrp:      Pointer to a buffer of size strlen(input) - 6. Will be
 *                 updated to contain the null-terminated human readable part.
 *       data:     Pointer to a buffer of size strlen(input) - 8 that will
 *                 hold the encoded 5-bit data values.
 *       data_len: Pointer to a size_t that will be updated to be the number
 *                 of entries in data.
 *  In: input:     Pointer to a null-terminated Bech32 string.
 *  Returns 1 if succesful.
 */
int bech32_decode(
        char *hrp,
        uint8_t *data,
        size_t *data_len,
        const char *input
);

int bech32_decode_nocheck(uint8_t *data, size_t *data_len, const char *input);

#endif
#include "Bloom.h"
#include <iostream>
#include <math.h>
#include <string.h>
//#include <unistd.h>

#define MAKESTRING(n) STRING(n)
#define STRING(n) #n
#define BLOOM_MAGIC "libbloom2"
#define BLOOM_VERSION_MAJOR 2
#define BLOOM_VERSION_MINOR 1

Bloom::Bloom(unsigned long long entries, double error) : _ready(0)
{
    if (entries < 1000 || error <= 0 || error >= 1) {
        printf("Bloom init error\n");
        return;
    }

    _entries = entries;
    _error = error;

    long double num = -log(_error);
    long double denom = 0.480453013918201; // ln(2)^2
    _bpe = (num / denom);

    long double dentries = (long double)_entries;
    long double allbits = dentries * _bpe;
    _bits = (unsigned long long int)allbits;

    if (_bits % 8) {
        _bytes = (unsigned long long int)(_bits / 8) + 1;
    } else {
        _bytes = (unsigned long long int) _bits / 8;
    }

    _hashes = (unsigned char)ceil(0.693147180559945 * _bpe);  // ln(2)

    _bf = (unsigned char *)calloc((unsigned long long int)_bytes, sizeof(unsigned char));
    if (_bf == NULL) {                                   // LCOV_EXCL_START
        printf("Bloom init error\n");
        return;
    }                                                          // LCOV_EXCL_STOP

    _ready = 1;

    _major = BLOOM_VERSION_MAJOR;
    _minor = BLOOM_VERSION_MINOR;

}
Bloom::~Bloom()
{
    if (_ready)
        free(_bf);
}

int Bloom::check(const void *buffer, int len)
{
    return bloom_check_add(buffer, len, 0);
}


int Bloom::add(const void *buffer, int len)
{
    return bloom_check_add(buffer, len, 1);
}


void Bloom::print()
{
    printf("Bloom at %p\n", (void *)this);
    if (!_ready) {
        printf(" *** NOT READY ***\n");
    }
    printf("  Version     : %d.%d\n", _major, _minor);
    printf("  Entries     : %llu\n", _entries);
    printf("  Error       : %1.10f\n", _error);
    printf("  Bits        : %llu\n", _bits);
    printf("  Bits/Elem   : %f\n", _bpe);
    printf("  Bytes       : %llu", _bytes);
    unsigned int KB = _bytes / 1024;
    unsigned int MB = KB / 1024;
    //printf(" (%u KB, %u MB)\n", KB, MB);
    printf(" (%u MB)\n", MB);
    printf("  Hash funcs  : %d\n", _hashes);
}


int Bloom::reset()
{
    if (!_ready)
        return 1;
    memset(_bf, 0, _bytes);
    return 0;
}


int Bloom::save(const char *filename)
{
//    if (filename == NULL || filename[0] == 0) {
//        return 1;
//    }

//    int fd = open(filename, O_WRONLY | O_CREAT, 0644);
//    if (fd < 0) {
//        return 1;
//    }

//    ssize_t out = write(fd, BLOOM_MAGIC, strlen(BLOOM_MAGIC));
//    if (out != strlen(BLOOM_MAGIC)) {
//        goto save_error;        // LCOV_EXCL_LINE
//    }

//    uint16_t size = sizeof(struct bloom);
//    out = write(fd, &size, sizeof(uint16_t));
//    if (out != sizeof(uint16_t)) {
//        goto save_error;        // LCOV_EXCL_LINE
//    }

//    out = write(fd, bloom, sizeof(struct bloom));
//    if (out != sizeof(struct bloom)) {
//        goto save_error;        // LCOV_EXCL_LINE
//    }

//    out = write(fd, _bf, _bytes);
//    if (out != _bytes) {
//        goto save_error;        // LCOV_EXCL_LINE
//    }

//    close(fd);
//    return 0;
//    // LCOV_EXCL_START
//save_error:
//    close(fd);
//    return 1;
//    // LCOV_EXCL_STOP
    return 0;
}


int Bloom::load(const char *filename)
{
//    int rv = 0;

//    if (filename == NULL || filename[0] == 0) {
//        return 1;
//    }
//    if (bloom == NULL) {
//        return 2;
//    }

//    memset(bloom, 0, sizeof(struct bloom));

//    int fd = open(filename, O_RDONLY);
//    if (fd < 0) {
//        return 3;
//    }

//    char line[30];
//    memset(line, 0, 30);
//    ssize_t in = read(fd, line, strlen(BLOOM_MAGIC));

//    if (in != strlen(BLOOM_MAGIC)) {
//        rv = 4;
//        goto load_error;
//    }

//    if (strncmp(line, BLOOM_MAGIC, strlen(BLOOM_MAGIC))) {
//        rv = 5;
//        goto load_error;
//    }

//    uint16_t size;
//    in = read(fd, &size, sizeof(uint16_t));
//    if (in != sizeof(uint16_t)) {
//        rv = 6;
//        goto load_error;
//    }

//    if (size != sizeof(struct bloom)) {
//        rv = 7;
//        goto load_error;
//    }

//    in = read(fd, bloom, sizeof(struct bloom));
//    if (in != sizeof(struct bloom)) {
//        rv = 8;
//        goto load_error;
//    }

//    _bf = NULL;
//    if (_major != BLOOM_VERSION_MAJOR) {
//        rv = 9;
//        goto load_error;
//    }

//    _bf = (unsigned char *)malloc(_bytes);
//    if (_bf == NULL) {
//        rv = 10;        // LCOV_EXCL_LINE
//        goto load_error;
//    }

//    in = read(fd, _bf, _bytes);
//    if (in != _bytes) {
//        rv = 11;
//        free(_bf);
//        _bf = NULL;
//        goto load_error;
//    }

//    close(fd);
//    return rv;

//load_error:
//    close(fd);
//    _ready = 0;
//    return rv;
    return 0;
}


unsigned char Bloom::get_hashes()
{
    return _hashes;
}
unsigned long long int Bloom::get_bits()
{
    return _bits;
}
unsigned long long int Bloom::get_bytes()
{
    return _bytes;
}
const unsigned char *Bloom::get_bf()
{
    return _bf;
}

int Bloom::test_bit_set_bit(unsigned char *buf, unsigned int bit, int set_bit)
{
    unsigned int byte = bit >> 3;
    unsigned char c = buf[byte];        // expensive memory access
    unsigned char mask = 1 << (bit % 8);

    if (c & mask) {
        return 1;
    } else {
        if (set_bit) {
            buf[byte] = c | mask;
        }
        return 0;
    }
}

int Bloom::bloom_check_add(const void *buffer, int len, int add)
{
    if (_ready == 0) {
        printf("bloom not initialized!\n");
        return -1;
    }

    unsigned char hits = 0;
    unsigned int a = murmurhash2(buffer, len, 0x9747b28c);
    unsigned int b = murmurhash2(buffer, len, a);
    unsigned int x;
    unsigned char i;

    for (i = 0; i < _hashes; i++) {
        x = (a + b * i) % _bits;
        if (test_bit_set_bit(_bf, x, add)) {
            hits++;
        } else if (!add) {
            // Don't care about the presence of all the bits. Just our own.
            return 0;
        }
    }

    if (hits == _hashes) {
        return 1;                // 1 == element already in (or collision)
    }

    return 0;
}

// MurmurHash2, by Austin Appleby

// Note - This code makes a few assumptions about how your machine behaves -

// 1. We can read a 4-byte value from any address without crashing
// 2. sizeof(int) == 4

// And it has a few limitations -

// 1. It will not work incrementally.
// 2. It will not produce the same results on little-endian and big-endian
//    machines.
unsigned int Bloom::murmurhash2(const void *key, int len, const unsigned int seed)
{
    // 'm' and 'r' are mixing constants generated offline.
    // They're not really 'magic', they just happen to work well.

    const unsigned int m = 0x5bd1e995;
    const int r = 24;

    // Initialize the hash to a 'random' value

    unsigned int h = seed ^ len;

    // Mix 4 bytes at a time into the hash

    const unsigned char *data = (const unsigned char *)key;

    while (len >= 4) {
        unsigned int k = *(unsigned int *)data;

        k *= m;
        k ^= k >> r;
        k *= m;

        h *= m;
        h ^= k;

        data += 4;
        len -= 4;
    }

    // Handle the last few bytes of the input array

    switch (len) {
    case 3: h ^= data[2] << 16;
    case 2: h ^= data[1] << 8;
    case 1: h ^= data[0];
        h *= m;
    };

    // Do a few final mixes of the hash to ensure the last few
    // bytes are well-incorporated.

    h ^= h >> 13;
    h *= m;
    h ^= h >> 15;

    return h;
}

#ifndef BLOOMFILTER_H
#define BLOOMFILTER_H


class Bloom
{
public:
    Bloom(unsigned long long int entries, double error);
    ~Bloom();
    int check(const void *buffer, int len);
    int add(const void *buffer, int len);
    void print();
    int reset();
    int save(const char *filename);
    int load(const char *filename);

    unsigned char get_hashes();
    unsigned long long int get_bits();
    unsigned long long int get_bytes();
    const unsigned char *get_bf();

private:
    static unsigned int murmurhash2(const void *key, int len, const unsigned int seed);
    int test_bit_set_bit(unsigned char *buf, unsigned int bit, int set_bit);
    int bloom_check_add(const void *buffer, int len, int add);

private:
    // These fields are part of the public interface of this structure.
    // Client code may read these values if desired. Client code MUST NOT
    // modify any of these.
    unsigned long long int _entries;
    unsigned long long int _bits;
    unsigned long long int _bytes;
    unsigned char _hashes;
    double _error;

    // Fields below are private to the implementation. These may go away or
    // change incompatibly at any moment. Client code MUST NOT access or rely
    // on these.
    unsigned char _ready;
    unsigned char _major;
    unsigned char _minor;
    double _bpe;
    unsigned char *_bf;
};

#endif // BLOOMFILTER_H
/*
 * This file is part of the BSGS distribution (https://github.com/JeanLucPons/Kangaroo).
 * Copyright (c) 2020 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#include "Int.h"
#include "IntGroup.h"
#include <string.h>
#include <math.h>
#include <emmintrin.h>
#include "Timer.h"

#define MAX(x,y) (((x)>(y))?(x):(y))
#define MIN(x,y) (((x)<(y))?(x):(y))

Int _ONE((uint64_t)1);


// ------------------------------------------------

Int::Int() {
}

Int::Int(Int* a) {
	if (a) Set(a);
	else CLEAR();
}

Int::Int(int64_t i64) {

	if (i64 < 0) {
		CLEARFF();
	}
	else {
		CLEAR();
	}
	bits64[0] = i64;

}

Int::Int(uint64_t u64) {

	CLEAR();
	bits64[0] = u64;

}

// ------------------------------------------------

void Int::CLEAR() {

	memset(bits64, 0, NB64BLOCK * 8);

}

void Int::CLEARFF() {

	memset(bits64, 0xFF, NB64BLOCK * 8);

}

// ------------------------------------------------

void Int::Set(Int* a) {

	for (int i = 0; i < NB64BLOCK; i++)
		bits64[i] = a->bits64[i];

}

// ------------------------------------------------

void Int::Add(Int* a) {

	unsigned char c = 0;
	c = _addcarry_u64(c, bits64[0], a->bits64[0], bits64 + 0);
	c = _addcarry_u64(c, bits64[1], a->bits64[1], bits64 + 1);
	c = _addcarry_u64(c, bits64[2], a->bits64[2], bits64 + 2);
	c = _addcarry_u64(c, bits64[3], a->bits64[3], bits64 + 3);
	c = _addcarry_u64(c, bits64[4], a->bits64[4], bits64 + 4);
#if NB64BLOCK > 5
	c = _addcarry_u64(c, bits64[5], a->bits64[5], bits64 + 5);
	c = _addcarry_u64(c, bits64[6], a->bits64[6], bits64 + 6);
	c = _addcarry_u64(c, bits64[7], a->bits64[7], bits64 + 7);
	c = _addcarry_u64(c, bits64[8], a->bits64[8], bits64 + 8);
#endif

}

// ------------------------------------------------

void Int::Add(uint64_t a) {

	unsigned char c = 0;
	c = _addcarry_u64(c, bits64[0], a, bits64 + 0);
	c = _addcarry_u64(c, bits64[1], 0, bits64 + 1);
	c = _addcarry_u64(c, bits64[2], 0, bits64 + 2);
	c = _addcarry_u64(c, bits64[3], 0, bits64 + 3);
	c = _addcarry_u64(c, bits64[4], 0, bits64 + 4);
#if NB64BLOCK > 5
	c = _addcarry_u64(c, bits64[5], 0, bits64 + 5);
	c = _addcarry_u64(c, bits64[6], 0, bits64 + 6);
	c = _addcarry_u64(c, bits64[7], 0, bits64 + 7);
	c = _addcarry_u64(c, bits64[8], 0, bits64 + 8);
#endif
}

// ------------------------------------------------
void Int::AddOne() {

	unsigned char c = 0;
	c = _addcarry_u64(c, bits64[0], 1, bits64 + 0);
	c = _addcarry_u64(c, bits64[1], 0, bits64 + 1);
	c = _addcarry_u64(c, bits64[2], 0, bits64 + 2);
	c = _addcarry_u64(c, bits64[3], 0, bits64 + 3);
	c = _addcarry_u64(c, bits64[4], 0, bits64 + 4);
#if NB64BLOCK > 5
	c = _addcarry_u64(c, bits64[5], 0, bits64 + 5);
	c = _addcarry_u64(c, bits64[6], 0, bits64 + 6);
	c = _addcarry_u64(c, bits64[7], 0, bits64 + 7);
	c = _addcarry_u64(c, bits64[8], 0, bits64 + 8);
#endif

}

// ------------------------------------------------

void Int::Add(Int* a, Int* b) {

	unsigned char c = 0;
	c = _addcarry_u64(c, b->bits64[0], a->bits64[0], bits64 + 0);
	c = _addcarry_u64(c, b->bits64[1], a->bits64[1], bits64 + 1);
	c = _addcarry_u64(c, b->bits64[2], a->bits64[2], bits64 + 2);
	c = _addcarry_u64(c, b->bits64[3], a->bits64[3], bits64 + 3);
	c = _addcarry_u64(c, b->bits64[4], a->bits64[4], bits64 + 4);
#if NB64BLOCK > 5
	c = _addcarry_u64(c, b->bits64[5], a->bits64[5], bits64 + 5);
	c = _addcarry_u64(c, b->bits64[6], a->bits64[6], bits64 + 6);
	c = _addcarry_u64(c, b->bits64[7], a->bits64[7], bits64 + 7);
	c = _addcarry_u64(c, b->bits64[8], a->bits64[8], bits64 + 8);
#endif

}

// ------------------------------------------------

bool Int::IsGreater(Int* a) {

	int i;

	for (i = NB64BLOCK - 1; i >= 0;) {
		if (a->bits64[i] != bits64[i])
			break;
		i--;
	}

	if (i >= 0) {
		return bits64[i] > a->bits64[i];
	}
	else {
		return false;
	}

}

// ------------------------------------------------

bool Int::IsLower(Int* a) {

	int i;

	for (i = NB64BLOCK - 1; i >= 0;) {
		if (a->bits64[i] != bits64[i])
			break;
		i--;
	}

	if (i >= 0) {
		return bits64[i] < a->bits64[i];
	}
	else {
		return false;
	}

}

// ------------------------------------------------

bool Int::IsGreaterOrEqual(Int* a) {

	Int p;
	p.Sub(this, a);
	return p.IsPositive();

}

// ------------------------------------------------

bool Int::IsLowerOrEqual(Int* a) {

	int i = NB64BLOCK - 1;

	while (i >= 0) {
		if (a->bits64[i] != bits64[i])
			break;
		i--;
	}

	if (i >= 0) {
		return bits64[i] < a->bits64[i];
	}
	else {
		return true;
	}

}

bool Int::IsEqual(Int* a) {

	return

#if NB64BLOCK > 5
		(bits64[8] == a->bits64[8]) &&
		(bits64[7] == a->bits64[7]) &&
		(bits64[6] == a->bits64[6]) &&
		(bits64[5] == a->bits64[5]) &&
#endif
		(bits64[4] == a->bits64[4]) &&
		(bits64[3] == a->bits64[3]) &&
		(bits64[2] == a->bits64[2]) &&
		(bits64[1] == a->bits64[1]) &&
		(bits64[0] == a->bits64[0]);

}

bool Int::IsOne() {
	return IsEqual(&_ONE);
}

bool Int::IsZero() {

#if NB64BLOCK > 5
	return (bits64[8] | bits64[7] | bits64[6] | bits64[5] | bits64[4] | bits64[3] | bits64[2] | bits64[1] | bits64[0]) == 0;
#else
	return (bits64[4] | bits64[3] | bits64[2] | bits64[1] | bits64[0]) == 0;
#endif

}


// ------------------------------------------------

void Int::SetInt32(uint32_t value) {

	CLEAR();
	bits[0] = value;

}

// ------------------------------------------------

uint32_t Int::GetInt32() {
	return bits[0];
}

// ------------------------------------------------

unsigned char Int::GetByte(int n) {

	unsigned char* bbPtr = (unsigned char*)bits;
	return bbPtr[n];

}

void Int::Set32Bytes(unsigned char* bytes) {

	CLEAR();
	uint64_t* ptr = (uint64_t*)bytes;
	bits64[3] = _byteswap_uint64(ptr[0]);
	bits64[2] = _byteswap_uint64(ptr[1]);
	bits64[1] = _byteswap_uint64(ptr[2]);
	bits64[0] = _byteswap_uint64(ptr[3]);

}

void Int::Get32Bytes(unsigned char* buff) {

	uint64_t* ptr = (uint64_t*)buff;
	ptr[3] = _byteswap_uint64(bits64[0]);
	ptr[2] = _byteswap_uint64(bits64[1]);
	ptr[1] = _byteswap_uint64(bits64[2]);
	ptr[0] = _byteswap_uint64(bits64[3]);

}

// ------------------------------------------------

void Int::SetByte(int n, unsigned char byte) {

	unsigned char* bbPtr = (unsigned char*)bits;
	bbPtr[n] = byte;

}

// ------------------------------------------------

void Int::SetDWord(int n, uint32_t b) {
	bits[n] = b;
}

// ------------------------------------------------

void Int::SetQWord(int n, uint64_t b) {
	bits64[n] = b;
}

// ------------------------------------------------

void Int::Sub(Int* a) {

	unsigned char c = 0;
	c = _subborrow_u64(c, bits64[0], a->bits64[0], bits64 + 0);
	c = _subborrow_u64(c, bits64[1], a->bits64[1], bits64 + 1);
	c = _subborrow_u64(c, bits64[2], a->bits64[2], bits64 + 2);
	c = _subborrow_u64(c, bits64[3], a->bits64[3], bits64 + 3);
	c = _subborrow_u64(c, bits64[4], a->bits64[4], bits64 + 4);
#if NB64BLOCK > 5
	c = _subborrow_u64(c, bits64[5], a->bits64[5], bits64 + 5);
	c = _subborrow_u64(c, bits64[6], a->bits64[6], bits64 + 6);
	c = _subborrow_u64(c, bits64[7], a->bits64[7], bits64 + 7);
	c = _subborrow_u64(c, bits64[8], a->bits64[8], bits64 + 8);
#endif

}

// ------------------------------------------------

void Int::Sub(Int* a, Int* b) {

	unsigned char c = 0;
	c = _subborrow_u64(c, a->bits64[0], b->bits64[0], bits64 + 0);
	c = _subborrow_u64(c, a->bits64[1], b->bits64[1], bits64 + 1);
	c = _subborrow_u64(c, a->bits64[2], b->bits64[2], bits64 + 2);
	c = _subborrow_u64(c, a->bits64[3], b->bits64[3], bits64 + 3);
	c = _subborrow_u64(c, a->bits64[4], b->bits64[4], bits64 + 4);
#if NB64BLOCK > 5
	c = _subborrow_u64(c, a->bits64[5], b->bits64[5], bits64 + 5);
	c = _subborrow_u64(c, a->bits64[6], b->bits64[6], bits64 + 6);
	c = _subborrow_u64(c, a->bits64[7], b->bits64[7], bits64 + 7);
	c = _subborrow_u64(c, a->bits64[8], b->bits64[8], bits64 + 8);
#endif

}

void Int::Sub(uint64_t a) {

	unsigned char c = 0;
	c = _subborrow_u64(c, bits64[0], a, bits64 + 0);
	c = _subborrow_u64(c, bits64[1], 0, bits64 + 1);
	c = _subborrow_u64(c, bits64[2], 0, bits64 + 2);
	c = _subborrow_u64(c, bits64[3], 0, bits64 + 3);
	c = _subborrow_u64(c, bits64[4], 0, bits64 + 4);
#if NB64BLOCK > 5
	c = _subborrow_u64(c, bits64[5], 0, bits64 + 5);
	c = _subborrow_u64(c, bits64[6], 0, bits64 + 6);
	c = _subborrow_u64(c, bits64[7], 0, bits64 + 7);
	c = _subborrow_u64(c, bits64[8], 0, bits64 + 8);
#endif

}

void Int::SubOne() {

	unsigned char c = 0;
	c = _subborrow_u64(c, bits64[0], 1, bits64 + 0);
	c = _subborrow_u64(c, bits64[1], 0, bits64 + 1);
	c = _subborrow_u64(c, bits64[2], 0, bits64 + 2);
	c = _subborrow_u64(c, bits64[3], 0, bits64 + 3);
	c = _subborrow_u64(c, bits64[4], 0, bits64 + 4);
#if NB64BLOCK > 5
	c = _subborrow_u64(c, bits64[5], 0, bits64 + 5);
	c = _subborrow_u64(c, bits64[6], 0, bits64 + 6);
	c = _subborrow_u64(c, bits64[7], 0, bits64 + 7);
	c = _subborrow_u64(c, bits64[8], 0, bits64 + 8);
#endif

}

// ------------------------------------------------

bool Int::IsPositive() {
	return (int64_t)(bits64[NB64BLOCK - 1]) >= 0;
}

// ------------------------------------------------

bool Int::IsNegative() {
	return (int64_t)(bits64[NB64BLOCK - 1]) < 0;
}

// ------------------------------------------------

bool Int::IsStrictPositive() {
	if (IsPositive())
		return !IsZero();
	else
		return false;
}

// ------------------------------------------------

bool Int::IsEven() {
	return (bits[0] & 0x1) == 0;
}

// ------------------------------------------------

bool Int::IsOdd() {
	return (bits[0] & 0x1) == 1;
}

// ------------------------------------------------

void Int::Neg() {

	volatile unsigned char c = 0;
	c = _subborrow_u64(c, 0, bits64[0], bits64 + 0);
	c = _subborrow_u64(c, 0, bits64[1], bits64 + 1);
	c = _subborrow_u64(c, 0, bits64[2], bits64 + 2);
	c = _subborrow_u64(c, 0, bits64[3], bits64 + 3);
	c = _subborrow_u64(c, 0, bits64[4], bits64 + 4);
#if NB64BLOCK > 5
	c = _subborrow_u64(c, 0, bits64[5], bits64 + 5);
	c = _subborrow_u64(c, 0, bits64[6], bits64 + 6);
	c = _subborrow_u64(c, 0, bits64[7], bits64 + 7);
	c = _subborrow_u64(c, 0, bits64[8], bits64 + 8);
#endif

}

// ------------------------------------------------

void Int::ShiftL32Bit() {

	for (int i = NB32BLOCK - 1; i > 0; i--) {
		bits[i] = bits[i - 1];
	}
	bits[0] = 0;

}

// ------------------------------------------------

void Int::ShiftL64Bit() {

	for (int i = NB64BLOCK - 1; i > 0; i--) {
		bits64[i] = bits64[i - 1];
	}
	bits64[0] = 0;

}

// ------------------------------------------------

void Int::ShiftL32BitAndSub(Int* a, int n) {

	Int b;
	int i = NB32BLOCK - 1;

	for (; i >= n; i--)
		b.bits[i] = ~a->bits[i - n];
	for (; i >= 0; i--)
		b.bits[i] = 0xFFFFFFFF;

	Add(&b);
	AddOne();

}

// ------------------------------------------------

void Int::ShiftL(uint32_t n) {

	if (n < 64) {
		shiftL((unsigned char)n, bits64);
	}
	else {
		uint32_t nb64 = n / 64;
		uint32_t nb = n % 64;
		for (uint32_t i = 0; i < nb64; i++) ShiftL64Bit();
		shiftL((unsigned char)nb, bits64);
	}

}

// ------------------------------------------------

void Int::ShiftR32Bit() {

	for (int i = 0; i < NB32BLOCK - 1; i++) {
		bits[i] = bits[i + 1];
	}
	if (((int32_t)bits[NB32BLOCK - 2]) < 0)
		bits[NB32BLOCK - 1] = 0xFFFFFFFF;
	else
		bits[NB32BLOCK - 1] = 0;

}

// ------------------------------------------------

void Int::ShiftR64Bit() {

	for (int i = 0; i < NB64BLOCK - 1; i++) {
		bits64[i] = bits64[i + 1];
	}
	if (((int64_t)bits64[NB64BLOCK - 2]) < 0)
		bits64[NB64BLOCK - 1] = 0xFFFFFFFFFFFFFFFF;
	else
		bits64[NB64BLOCK - 1] = 0;

}

// ---------------------------------D---------------

void Int::ShiftR(uint32_t n) {

	if (n < 64) {
		shiftR((unsigned char)n, bits64);
	}
	else {
		uint32_t nb64 = n / 64;
		uint32_t nb = n % 64;
		for (uint32_t i = 0; i < nb64; i++) ShiftR64Bit();
		shiftR((unsigned char)nb, bits64);
	}

}

// ------------------------------------------------

void Int::SwapBit(int bitNumber) {

	uint32_t nb64 = bitNumber / 64;
	uint32_t nb = bitNumber % 64;
	uint64_t mask = 1ULL << nb;
	if (bits64[nb64] & mask) {
		bits64[nb64] &= ~mask;
	}
	else {
		bits64[nb64] |= mask;
	}

}

// ------------------------------------------------

void Int::Mult(Int* a) {

	Int b(this);
	Mult(a, &b);

}

// ------------------------------------------------

void Int::IMult(int64_t a) {

	// Make a positive
	if (a < 0LL) {
		a = -a;
		Neg();
	}

	imm_mul(bits64, a, bits64);

}

// ------------------------------------------------

void Int::Mult(uint64_t a) {

	imm_mul(bits64, a, bits64);

}
// ------------------------------------------------

void Int::IMult(Int* a, int64_t b) {

	Set(a);

	// Make b positive
	if (b < 0LL) {
		Neg();
		b = -b;
	}
	imm_mul(bits64, b, bits64);

}

// ------------------------------------------------

void Int::Mult(Int* a, uint64_t b) {

	imm_mul(a->bits64, b, bits64);

}

// ------------------------------------------------

void Int::Mult(Int* a, Int* b) {

	unsigned char c = 0;
	uint64_t h;
	uint64_t pr = 0;
	uint64_t carryh = 0;
	uint64_t carryl = 0;

	bits64[0] = _umul128(a->bits64[0], b->bits64[0], &pr);

	for (int i = 1; i < NB64BLOCK; i++) {
		for (int j = 0; j <= i; j++) {
			c = _addcarry_u64(c, _umul128(a->bits64[j], b->bits64[i - j], &h), pr, &pr);
			c = _addcarry_u64(c, carryl, h, &carryl);
			c = _addcarry_u64(c, carryh, 0, &carryh);
		}
		bits64[i] = pr;
		pr = carryl;
		carryl = carryh;
		carryh = 0;
	}

}

// ------------------------------------------------

void Int::Mult(Int* a, uint32_t b) {
	imm_mul(a->bits64, (uint64_t)b, bits64);
}

// ------------------------------------------------

double Int::ToDouble() {

	double base = 1.0;
	double sum = 0;
	double pw32 = pow(2.0, 32.0);
	for (int i = 0; i < NB32BLOCK; i++) {
		sum += (double)(bits[i]) * base;
		base *= pw32;
	}

	return sum;

}

// ------------------------------------------------

static uint32_t bitLength(uint32_t dw) {

	uint32_t mask = 0x80000000;
	uint32_t b = 0;
	while (b < 32 && (mask & dw) == 0) {
		b++;
		mask >>= 1;
	}
	return b;

}

// ------------------------------------------------

int Int::GetBitLength() {

	Int t(this);
	if (IsNegative())
		t.Neg();

	int i = NB32BLOCK - 1;
	while (i >= 0 && t.bits[i] == 0) i--;
	if (i < 0) return 0;
	return (32 - bitLength(t.bits[i])) + i * 32;

}

// ------------------------------------------------

int Int::GetSize() {

	int i = NB32BLOCK - 1;
	while (i > 0 && bits[i] == 0) i--;
	return i + 1;

}

// ------------------------------------------------

void Int::MultModN(Int* a, Int* b, Int* n) {

	Int r;
	Mult(a, b);
	Div(n, &r);
	Set(&r);

}

// ------------------------------------------------

void Int::Mod(Int* n) {

	Int r;
	Div(n, &r);
	Set(&r);

}

// ------------------------------------------------

int Int::GetLowestBit() {

	// Assume this!=0
	int b = 0;
	while (GetBit(b) == 0) b++;
	return b;

}

// ------------------------------------------------

void Int::MaskByte(int n) {

	for (int i = n; i < NB32BLOCK; i++)
		bits[i] = 0;

}

// ------------------------------------------------

void Int::Abs() {

	if (IsNegative())
		Neg();

}

// ------------------------------------------------

void Int::Rand(int nbit) {

	CLEAR();

	uint32_t nb = nbit / 32;
	uint32_t leftBit = nbit % 32;
	uint32_t mask = 1;
	mask = (mask << leftBit) - 1;
	uint32_t i = 0;
	for (; i < nb; i++)
		bits[i] = rndl();
	bits[i] = rndl() & mask;

}

// ------------------------------------------------

void Int::Rand(Int* randMax) {

	int b = randMax->GetBitLength();
	Int r;
	r.Rand(b);
	Int q(&r);
	Int rem;
	q.Div(randMax, &rem);
	Set(&rem);

}

// ------------------------------------------------

void Int::Div(Int* a, Int* mod) {

	if (a->IsGreater(this)) {
		if (mod) mod->Set(this);
		CLEAR();
		return;
	}

	if (a->IsZero()) {
		printf("Divide by 0!\n");
		return;
	}

	if (IsEqual(a)) {
		if (mod) mod->CLEAR();
		Set(&_ONE);
		return;
	}

	//Division algorithm D (Knuth section 4.3.1)

	Int rem(this);
	Int d(a);
	Int dq;
	CLEAR();

	// Size
	uint32_t dSize = d.GetSize();
	uint32_t tSize = rem.GetSize();
	uint32_t qSize = tSize - dSize + 1;

	// D1 normalize the divisor
	uint32_t shift = bitLength(d.bits[dSize - 1]);
	if (shift > 0) {
		d.ShiftL(shift);
		rem.ShiftL(shift);
	}

	uint32_t  _dh = d.bits[dSize - 1];
	uint64_t  dhLong = _dh;
	uint32_t  _dl = (dSize > 1) ? d.bits[dSize - 2] : 0;
	int sb = tSize - 1;

	// D2 Initialize j
	for (int j = 0; j < (int)qSize; j++) {

		// D3 Estimate qhat
		uint32_t qhat = 0;
		uint32_t qrem = 0;
		int skipCorrection = false;
		uint32_t nh = rem.bits[sb - j + 1];
		uint32_t nm = rem.bits[sb - j];

		if (nh == _dh) {
			qhat = ~0;
			qrem = nh + nm;
			skipCorrection = qrem < nh;
		}
		else {
			uint64_t nChunk = ((uint64_t)nh << 32) | (uint64_t)nm;
			qhat = (uint32_t)(nChunk / dhLong);
			qrem = (uint32_t)(nChunk % dhLong);
		}

		if (qhat == 0)
			continue;

		if (!skipCorrection) {

			// Correct qhat
			uint64_t nl = (uint64_t)rem.bits[sb - j - 1];
			uint64_t rs = ((uint64_t)qrem << 32) | nl;
			uint64_t estProduct = (uint64_t)_dl * (uint64_t)(qhat);

			if (estProduct > rs) {
				qhat--;
				qrem = (uint32_t)(qrem + (uint32_t)dhLong);
				if ((uint64_t)qrem >= dhLong) {
					estProduct = (uint64_t)_dl * (uint64_t)(qhat);
					rs = ((uint64_t)qrem << 32) | nl;
					if (estProduct > rs)
						qhat--;
				}
			}

		}

		// D4 Multiply and subtract    
		dq.Mult(&d, qhat);
		rem.ShiftL32BitAndSub(&dq, qSize - j - 1);
		if (rem.IsNegative()) {
			// Overflow
			rem.Add(&d);
			qhat--;
		}

		bits[qSize - j - 1] = qhat;

	}

	if (mod) {
		// Unnormalize remainder
		rem.ShiftR(shift);
		mod->Set(&rem);
	}

}

// ------------------------------------------------

void Int::GCD(Int* a) {

	uint32_t k;
	uint32_t b;

	Int U(this);
	Int V(a);
	Int T;

	if (U.IsZero()) {
		Set(&V);
		return;
	}

	if (V.IsZero()) {
		Set(&U);
		return;
	}

	if (U.IsNegative()) U.Neg();
	if (V.IsNegative()) V.Neg();

	k = 0;
	while (U.GetBit(k) == 0 && V.GetBit(k) == 0)
		k++;
	U.ShiftR(k);
	V.ShiftR(k);
	if (U.GetBit(0) == 1) {
		T.Set(&V);
		T.Neg();
	}
	else {
		T.Set(&U);
	}

	do {

		if (T.IsNegative()) {
			T.Neg();
			b = 0; while (T.GetBit(b) == 0) b++;
			T.ShiftR(b);
			V.Set(&T);
			T.Set(&U);
		}
		else {
			b = 0; while (T.GetBit(b) == 0) b++;
			T.ShiftR(b);
			U.Set(&T);
		}

		T.Sub(&V);

	} while (!T.IsZero());

	// Store gcd
	Set(&U);
	ShiftL(k);

}

// ------------------------------------------------

void Int::SetBase10(char* value) {

	CLEAR();
	Int pw((uint64_t)1);
	Int c;
	int lgth = (int)strlen(value);
	for (int i = lgth - 1; i >= 0; i--) {
		uint32_t id = (uint32_t)(value[i] - '0');
		c.Set(&pw);
		c.Mult(id);
		Add(&c);
		pw.Mult(10);
	}

}

// ------------------------------------------------

void  Int::SetBase16(char* value) {
	SetBaseN(16, "0123456789ABCDEF", value);
}

// ------------------------------------------------

std::string Int::GetBase10() {
	return GetBaseN(10, "0123456789");
}

// ------------------------------------------------

std::string Int::GetBase16() {
	return GetBaseN(16, "0123456789ABCDEF");
}

// ------------------------------------------------

std::string Int::GetBlockStr() {

	char tmp[256];
	char bStr[256];
	tmp[0] = 0;
	for (int i = NB32BLOCK - 3; i >= 0; i--) {
		sprintf(bStr, "%08X", bits[i]);
		strcat(tmp, bStr);
		if (i != 0) strcat(tmp, " ");
	}
	return std::string(tmp);
}

// ------------------------------------------------

std::string Int::GetC64Str(int nbDigit) {

	char tmp[256];
	char bStr[256];
	tmp[0] = '{';
	tmp[1] = 0;
	for (int i = 0; i < nbDigit; i++) {
		if (bits64[i] != 0) {
#ifdef WIN64
			sprintf(bStr, "0x%016I64XULL", bits64[i]);
#else
			sprintf(bStr, "0x%" PRIx64  "ULL", bits64[i]);
#endif
		}
		else {
			sprintf(bStr, "0ULL");
		}
		strcat(tmp, bStr);
		if (i != nbDigit - 1) strcat(tmp, ",");
	}
	strcat(tmp, "}");
	return std::string(tmp);
}

// ------------------------------------------------

void  Int::SetBaseN(int n, char* charset, char* value) {

	CLEAR();

	Int pw((uint64_t)1);
	Int nb((uint64_t)n);
	Int c;

	int lgth = (int)strlen(value);
	for (int i = lgth - 1; i >= 0; i--) {
		char* p = strchr(charset, toupper(value[i]));
		if (!p) {
			printf("Invalid charset !!\n");
			return;
		}
		int id = (int)(p - charset);
		c.SetInt32(id);
		c.Mult(&pw);
		Add(&c);
		pw.Mult(&nb);

	}

}

// ------------------------------------------------

std::string Int::GetBaseN(int n, char* charset) {

	std::string ret;

	Int N(this);
	int isNegative = N.IsNegative();
	if (isNegative) N.Neg();

	// TODO: compute max digit
	unsigned char digits[1024];
	memset(digits, 0, sizeof(digits));

	int digitslen = 1;
	for (int i = 0; i < NB64BLOCK * 8; i++) {
		unsigned int carry = N.GetByte(NB64BLOCK * 8 - i - 1);
		for (int j = 0; j < digitslen; j++) {
			carry += (unsigned int)(digits[j]) << 8;
			digits[j] = (unsigned char)(carry % n);
			carry /= n;
		}
		while (carry > 0) {
			digits[digitslen++] = (unsigned char)(carry % n);
			carry /= n;
		}
	}

	// reverse
	if (isNegative)
		ret.push_back('-');

	for (int i = 0; i < digitslen; i++)
		ret.push_back(charset[digits[digitslen - 1 - i]]);

	if (ret.length() == 0)
		ret.push_back('0');

	return ret;

}

// ------------------------------------------------


int Int::GetBit(uint32_t n) {

	uint32_t byte = n >> 5;
	uint32_t bit = n & 31;
	uint32_t mask = 1 << bit;
	return (bits[byte] & mask) != 0;

}

// ------------------------------------------------

std::string Int::GetBase2() {

	char ret[1024];
	int k = 0;

	for (int i = 0; i < NB32BLOCK - 1; i++) {
		unsigned int mask = 0x80000000;
		for (int j = 0; j < 32; j++) {
			if (bits[i] & mask) ret[k] = '1';
			else             ret[k] = '0';
			k++;
			mask = mask >> 1;
		}
	}
	ret[k] = 0;

	return std::string(ret);

}

bool Int::IsProbablePrime() {

	// Prime cheking (probalistic Miller-Rabin test)
	Int::SetupField(this);
	int nbBit = GetBitLength();

	Int Q(this);
	Q.SubOne();
	Int N1(&Q);
	uint64_t e = 0;
	while (Q.IsEven()) {
		Q.ShiftR(1);
		e++;
	}

	uint64_t k = 50;

	for (uint64_t i = 0; i < k; i++) {

		Int a;
		Int x;
		x.SetInt32(0);
		while (x.IsLowerOrEqual(&_ONE) || x.IsGreaterOrEqual(&N1))
			x.Rand(nbBit);
		x.ModExp(&Q);
		if (x.IsOne() || x.IsEqual(&N1))
			continue;

		for (uint64_t j = 0; j < e - 1; j++) {
			x.ModSquare(&x);
			if (x.IsOne()) {
				// Composite
				return false;
			}
			if (x.IsEqual(&N1))
				break;
		}

		if (x.IsEqual(&N1))
			continue;

		return false;

	}

	// Probable prime
	return true;

}


// ------------------------------------------------

void Int::Check() {

	double t0;
	double t1;
	double tTotal;
	int   i;
	bool ok;

	Int a, b, c, d, e, R;

	a.SetBase10("4743256844168384767987");
	b.SetBase10("1679314142928575978367");
	if (strcmp(a.GetBase10().c_str(), "4743256844168384767987") != 0) {
		printf(" GetBase10() failed ! %s!=4743256844168384767987\n", a.GetBase10().c_str());
	}
	if (strcmp(b.GetBase10().c_str(), "1679314142928575978367") != 0) {
		printf(" GetBase10() failed ! %s!=1679314142928575978367\n", b.GetBase10().c_str());
		return;
	}

	printf("GetBase10() Results OK\n");

	// Add -------------------------------------------------------------------------------------------
	t0 = Timer::get_tick();
	for (i = 0; i < 10000; i++) c.Add(&a, &b);
	t1 = Timer::get_tick();

	if (c.GetBase10() == "6422570987096960746354") {
		printf("Add() Results OK : ");
		Timer::printResult("Add", 10000, t0, t1);
	}
	else {
		printf("Add() Results Wrong\nR=%s\nT=6422570987096960746354\n", c.GetBase10().c_str());
		return;
	}

	// Mult -------------------------------------------------------------------------------------------
	a.SetBase10("3890902718436931151119442452387018319292503094706912504064239834754167");
	b.SetBase10("474325684416838476798716793141429285759783676422570987096960746354");
	e.SetBase10("1845555094921934741640873731771879197054909502699192730283220486240724687661257894226660948002650341240452881231721004292250660431557118");

	t0 = Timer::get_tick();
	for (i = 0; i < 10000; i++) c.Mult(&a, &b);
	t1 = Timer::get_tick();

	if (c.IsEqual(&e)) {
		printf("Mult() Results OK : ");
		Timer::printResult("Mult", 10000, t0, t1);
	}
	else {
		printf("Mult() Results Wrong\nR=%s\nT=%s\n", e.GetBase10().c_str(), c.GetBase10().c_str());
		return;
	}

	// Div -------------------------------------------------------------------------------------------
	tTotal = 0.0;
	ok = true;
	for (int i = 0; i < 1000 && ok; i++) {

		a.Rand(BISIZE);
		b.Rand(BISIZE / 2);
		d.Set(&a);
		e.Set(&b);

		t0 = Timer::get_tick();
		a.Div(&b, &c);
		t1 = Timer::get_tick();
		tTotal += (t1 - t0);

		a.Mult(&e);
		a.Add(&c);
		if (!a.IsEqual(&d)) {
			ok = false;
			printf("Div() Results Wrong \nN: %s\nD: %s\nQ: %s\nR: %s\n",
				d.GetBase16().c_str(),
				b.GetBase16().c_str(),
				a.GetBase16().c_str(),
				c.GetBase16().c_str()

			);
			return;
		}

	}

	if (ok) {
		printf("Div() Results OK : ");
		Timer::printResult("Div", 1000, 0, tTotal);
	}

	// Modular arithmetic -------------------------------------------------------------------------------
	// SecpK1 prime
	b.SetBase16("FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F");
	Int::SetupField(&b);

	// ModInv -------------------------------------------------------------------------------------------

	for (int i = 0; i < 1000 && ok; i++) {
		a.Rand(BISIZE);
		b = a;
		a.ModInv();
		a.ModMul(&b);
		if (!a.IsOne()) {
			printf("ModInv() Results Wrong [%d] %s\n", i, a.GetBase16().c_str());
			ok = false;
		}
	}

	ok = true;
	for (int i = 0; i < 100 && ok; i++) {

		// Euler a^-1 = a^(p-2) mod p (p is prime)
		Int e(Int::GetFieldCharacteristic());
		e.Sub(2ULL);
		a.Rand(BISIZE);
		b = a;
		b.ModExp(&e);

		a.ModInv();
		if (!a.IsEqual(&b)) {
			ok = false;
		}

	}

	if (!ok) {
		printf("ModInv()/ModExp() Results Wrong:\nModInv=%s\nModExp=%s\n", a.GetBase16().c_str(), b.GetBase16().c_str());
		return;
	}
	else {
		printf("ModInv()/ModExp() Results OK\n");
	}

	t0 = Timer::get_tick();
	for (int i = 0; i < 100000; i++) {
		a.Rand(BISIZE);
		a.ModInv();
	}
	t1 = Timer::get_tick();

	printf("ModInv() Results OK : ");
	Timer::printResult("Inv", 100000, 0, t1 - t0);

	// IntGroup -----------------------------------------------------------------------------------

	Int m[256];
	Int chk[256];
	IntGroup g(256);

	g.Set(m);
	for (int i = 0; i < 256; i++) {
		m[i].Rand(256);
		chk[i].Set(m + i);
		chk[i].ModInv();
	}
	g.ModInv();
	ok = true;
	for (int i = 0; i < 256; i++) {
		if (!m[i].IsEqual(chk + i)) {
			ok = false;
			printf("IntGroup.ModInv() Wrong !\n");
			printf("[%d] %s\n", i, m[i].GetBase16().c_str());
			printf("[%d] %s\n", i, chk[i].GetBase16().c_str());
			return;
		}
	}

	t0 = Timer::get_tick();
	for (int j = 0; j < 1000; j++) {
		for (int i = 0; i < 256; i++) {
			m[i].Rand(256);
		}
		g.ModInv();
	}
	t1 = Timer::get_tick();

	printf("IntGroup.ModInv() Results OK : ");
	Timer::printResult("Inv", 1000 * 256, 0, t1 - t0);

	// ModMulK1 ------------------------------------------------------------------------------------

	for (int i = 0; i < 100000; i++) {
		a.Rand(BISIZE);
		b.Rand(BISIZE);
		c.ModMul(&a, &b);
		d.ModMulK1(&a, &b);
		if (!c.IsEqual(&d)) {
			printf("ModMulK1() Wrong !\n");
			printf("[%d] %s\n", i, c.GetBase16().c_str());
			printf("[%d] %s\n", i, d.GetBase16().c_str());
			return;
		}
	}

	t0 = Timer::get_tick();
	for (int i = 0; i < 1000000; i++) {
		a.Rand(BISIZE);
		b.Rand(BISIZE);
		c.ModMulK1(&a, &b);
	}
	t1 = Timer::get_tick();

	printf("ModMulK1() Results OK : ");
	Timer::printResult("Mult", 1000000, 0, t1 - t0);

	// ModSqrK1 ------------------------------------------------------------------------------------

	for (int i = 0; i < 100000; i++) {
		a.Rand(BISIZE);
		c.ModMul(&a, &a);
		d.ModSquareK1(&a);
		if (!c.IsEqual(&d)) {
			printf("ModSquareK1() Wrong !\n");
			printf("[%d] %s\n", i, c.GetBase16().c_str());
			printf("[%d] %s\n", i, d.GetBase16().c_str());
			return;
		}
	}

	t0 = Timer::get_tick();
	for (int i = 0; i < 1000000; i++) {
		a.Rand(BISIZE);
		b.Rand(BISIZE);
		c.ModSquareK1(&b);
	}
	t1 = Timer::get_tick();

	printf("ModSquareK1() Results OK : ");
	Timer::printResult("Mult", 1000000, 0, t1 - t0);

	// ModSqrt ------------------------------------------------------------------------------------
	b.SetBase16("FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F");
	Int::SetupField(&b);

	ok = true;
	for (int i = 0; i < 100 && ok; i++) {

		bool hasSqrt = false;
		while (!hasSqrt) {
			a.Rand(BISIZE);
			hasSqrt = !a.IsZero() && a.IsLower(Int::GetFieldCharacteristic()) && a.HasSqrt();
		}

		c.Set(&a);
		a.ModSqrt();
		b.ModSquare(&a);
		if (!b.IsEqual(&c)) {
			printf("ModSqrt() wrong !\n");
			ok = false;
		}

	}
	if (!ok) return;

	printf("ModSqrt() Results OK !\n");

	// ModMulK1 order -----------------------------------------------------------------------------
	// InitK1() is done by secpK1
	b.SetBase16("FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141");
	Int::SetupField(&b);
	Int::InitK1(&b);

	for (int i = 0; i < 100000; i++) {
		a.Rand(BISIZE);
		b.Rand(BISIZE);
		c.ModMul(&a, &b);
		d.Set(&a);
		d.ModMulK1order(&b);
		if (!c.IsEqual(&d)) {
			printf("ModMulK1order() Wrong !\n");
			printf("[%d] %s\n", i, c.GetBase16().c_str());
			printf("[%d] %s\n", i, d.GetBase16().c_str());
			return;
		}
	}

	t0 = Timer::get_tick();
	for (int i = 0; i < 1000000; i++) {
		a.Rand(BISIZE);
		b.Rand(BISIZE);
		c.Set(&a);
		c.ModMulK1order(&b);
	}
	t1 = Timer::get_tick();

	printf("ModMulK1order() Results OK : ");
	Timer::printResult("Mult", 1000000, 0, t1 - t0);

}
/*
 * This file is part of the BSGS distribution (https://github.com/JeanLucPons/Kangaroo).
 * Copyright (c) 2020 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

// Big integer class (Fixed size)

#ifndef BIGINTH
#define BIGINTH

#include "Random.h"
#include <string>
#include <inttypes.h>

// We need 1 extra block for Knuth div algorithm , Montgomery multiplication and ModInv
#define BISIZE 256

#if BISIZE==256
#define NB64BLOCK 5
#define NB32BLOCK 10
#elif BISIZE==512
#define NB64BLOCK 9
#define NB32BLOCK 18
#else
#error Unsuported size
#endif

class Int {

public:

	Int();
	Int(int64_t i64);
	Int(uint64_t u64);
	Int(Int* a);

	// Op
	void Add(uint64_t a);
	void Add(Int* a);
	void Add(Int* a, Int* b);
	void AddOne();
	void Sub(uint64_t a);
	void Sub(Int* a);
	void Sub(Int* a, Int* b);
	void SubOne();
	void Mult(Int* a);
	void Mult(uint64_t a);
	void IMult(int64_t a);
	void Mult(Int* a, uint64_t b);
	void IMult(Int* a, int64_t b);
	void Mult(Int* a, Int* b);
	void Div(Int* a, Int* mod = NULL);
	void MultModN(Int* a, Int* b, Int* n);
	void Neg();
	void Abs();

	// Right shift (signed)
	void ShiftR(uint32_t n);
	void ShiftR32Bit();
	void ShiftR64Bit();
	// Left shift
	void ShiftL(uint32_t n);
	void ShiftL32Bit();
	void ShiftL64Bit();
	// Bit swap
	void SwapBit(int bitNumber);


	// Comp 
	bool IsGreater(Int* a);
	bool IsGreaterOrEqual(Int* a);
	bool IsLowerOrEqual(Int* a);
	bool IsLower(Int* a);
	bool IsEqual(Int* a);
	bool IsZero();
	bool IsOne();
	bool IsStrictPositive();
	bool IsPositive();
	bool IsNegative();
	bool IsEven();
	bool IsOdd();
	bool IsProbablePrime();


	double ToDouble();

	// Modular arithmetic

	// Setup field
	// n is the field characteristic
	// R used in Montgomery mult (R = 2^size(n))
	// R2 = R^2, R3 = R^3, R4 = R^4
	static void SetupField(Int* n, Int* R = NULL, Int* R2 = NULL, Int* R3 = NULL, Int* R4 = NULL);
	static Int* GetR();                            // Return R
	static Int* GetR2();                           // Return R2
	static Int* GetR3();                           // Return R3
	static Int* GetR4();                           // Return R4
	static Int* GetFieldCharacteristic();          // Return field characteristic

	void GCD(Int* a);                          // this <- GCD(this,a)
	void Mod(Int* n);                          // this <- this (mod n)
	void ModInv();                             // this <- this^-1 (mod n)
	void MontgomeryMult(Int* a, Int* b);        // this <- a*b*R^-1 (mod n)
	void MontgomeryMult(Int* a);               // this <- this*a*R^-1 (mod n)
	void ModAdd(Int* a);                       // this <- this+a (mod n) [0<a<P]
	void ModAdd(Int* a, Int* b);                // this <- a+b (mod n) [0<a,b<P]
	void ModAdd(uint64_t a);                   // this <- this+a (mod n) [0<a<P]
	void ModSub(Int* a);                       // this <- this-a (mod n) [0<a<P]
	void ModSub(Int* a, Int* b);               // this <- a-b (mod n) [0<a,b<P]
	void ModSub(uint64_t a);                   // this <- this-a (mod n) [0<a<P]
	void ModMul(Int* a, Int* b);                // this <- a*b (mod n) 
	void ModMul(Int* a);                       // this <- this*b (mod n) 
	void ModSquare(Int* a);                    // this <- a^2 (mod n) 
	void ModCube(Int* a);                      // this <- a^3 (mod n) 
	void ModDouble();                          // this <- 2*this (mod n) 
	void ModExp(Int* e);                       // this <- this^e (mod n) 
	void ModNeg();                             // this <- -this (mod n)
	void ModSqrt();                            // this <- +/-sqrt(this) (mod n)
	bool HasSqrt();                            // true if this admit a square root

	// Specific SecpK1
	static void InitK1(Int* order);
	void ModMulK1(Int* a, Int* b);
	void ModMulK1(Int* a);
	void ModSquareK1(Int* a);
	void ModMulK1order(Int* a);
	void ModAddK1order(Int* a, Int* b);
	void ModAddK1order(Int* a);
	void ModSubK1order(Int* a);
	void ModNegK1order();
	uint32_t ModPositiveK1();

	// Size
	int GetSize();
	int GetBitLength();

	// Setter
	void SetInt32(uint32_t value);
	void Set(Int* a);
	void SetBase10(char* value);
	void SetBase16(char* value);
	void SetBaseN(int n, char* charset, char* value);
	void SetByte(int n, unsigned char byte);
	void SetDWord(int n, uint32_t b);
	void SetQWord(int n, uint64_t b);
	void Rand(int nbit);
	void Rand(Int* randMax);
	void Set32Bytes(unsigned char* bytes);
	void MaskByte(int n);

	// Getter
	uint32_t GetInt32();
	int GetBit(uint32_t n);
	unsigned char GetByte(int n);
	void Get32Bytes(unsigned char* buff);

	// To String
	std::string GetBase2();
	std::string GetBase10();
	std::string GetBase16();
	std::string GetBaseN(int n, char* charset);
	std::string GetBlockStr();
	std::string GetC64Str(int nbDigit);

	// Check function
	static void Check();


	/*
	// Align to 16 bytes boundary
	union {
	  __declspec(align(16)) uint32_t bits[NB32BLOCK];
	  __declspec(align(16)) uint64_t bits64[NB64BLOCK];
	};
	*/
	union {
		uint32_t bits[NB32BLOCK];
		uint64_t bits64[NB64BLOCK];
	};

private:

	void ShiftL32BitAndSub(Int* a, int n);
	uint64_t AddC(Int* a);
	void AddAndShift(Int* a, Int* b, uint64_t cH);
	void Mult(Int* a, uint32_t b);
	int  GetLowestBit();
	void CLEAR();
	void CLEARFF();


};

// Inline routines

#ifndef WIN64

// Missing intrinsics
static uint64_t inline _umul128(uint64_t a, uint64_t b, uint64_t* h) {
	uint64_t rhi;
	uint64_t rlo;
	__asm__("mulq  %[b];" :"=d"(rhi), "=a"(rlo) : "1"(a), [b]"rm"(b));
	*h = rhi;
	return rlo;
}

static uint64_t inline __shiftright128(uint64_t a, uint64_t b, unsigned char n) {
	uint64_t c;
	__asm__("movq %1,%0;shrdq %3,%2,%0;" : "=D"(c) : "r"(a), "r"(b), "c"(n));
	return  c;
}


static uint64_t inline __shiftleft128(uint64_t a, uint64_t b, unsigned char n) {
	uint64_t c;
	__asm__("movq %1,%0;shldq %3,%2,%0;" : "=D"(c) : "r"(b), "r"(a), "c"(n));
	return  c;
}

#define _subborrow_u64(a,b,c,d) __builtin_ia32_sbb_u64(a,b,c,(long long unsigned int*)d);
#define _addcarry_u64(a,b,c,d) __builtin_ia32_addcarryx_u64(a,b,c,(long long unsigned int*)d);
#define _byteswap_uint64 __builtin_bswap64
#else
#include <intrin.h>
#endif

static void inline imm_mul(uint64_t* x, uint64_t y, uint64_t* dst) {

	unsigned char c = 0;
	uint64_t h, carry;
	dst[0] = _umul128(x[0], y, &h); carry = h;
	c = _addcarry_u64(c, _umul128(x[1], y, &h), carry, dst + 1); carry = h;
	c = _addcarry_u64(c, _umul128(x[2], y, &h), carry, dst + 2); carry = h;
	c = _addcarry_u64(c, _umul128(x[3], y, &h), carry, dst + 3); carry = h;
	c = _addcarry_u64(c, _umul128(x[4], y, &h), carry, dst + 4); carry = h;
#if NB64BLOCK > 5
	c = _addcarry_u64(c, _umul128(x[5], y, &h), carry, dst + 5); carry = h;
	c = _addcarry_u64(c, _umul128(x[6], y, &h), carry, dst + 6); carry = h;
	c = _addcarry_u64(c, _umul128(x[7], y, &h), carry, dst + 7); carry = h;
	c = _addcarry_u64(c, _umul128(x[8], y, &h), carry, dst + 8); carry = h;
#endif

}

static void inline imm_umul(uint64_t* x, uint64_t y, uint64_t* dst) {

	// Assume that x[NB64BLOCK-1] is 0
	unsigned char c = 0;
	uint64_t h, carry;
	dst[0] = _umul128(x[0], y, &h); carry = h;
	c = _addcarry_u64(c, _umul128(x[1], y, &h), carry, dst + 1); carry = h;
	c = _addcarry_u64(c, _umul128(x[2], y, &h), carry, dst + 2); carry = h;
	c = _addcarry_u64(c, _umul128(x[3], y, &h), carry, dst + 3); carry = h;
#if NB64BLOCK > 5
	c = _addcarry_u64(c, _umul128(x[4], y, &h), carry, dst + 4); carry = h;
	c = _addcarry_u64(c, _umul128(x[5], y, &h), carry, dst + 5); carry = h;
	c = _addcarry_u64(c, _umul128(x[6], y, &h), carry, dst + 6); carry = h;
	c = _addcarry_u64(c, _umul128(x[7], y, &h), carry, dst + 7); carry = h;
#endif
	_addcarry_u64(c, 0ULL, carry, dst + (NB64BLOCK - 1));

}

static void inline shiftR(unsigned char n, uint64_t* d) {

	d[0] = __shiftright128(d[0], d[1], n);
	d[1] = __shiftright128(d[1], d[2], n);
	d[2] = __shiftright128(d[2], d[3], n);
	d[3] = __shiftright128(d[3], d[4], n);
#if NB64BLOCK > 5
	d[4] = __shiftright128(d[4], d[5], n);
	d[5] = __shiftright128(d[5], d[6], n);
	d[6] = __shiftright128(d[6], d[7], n);
	d[7] = __shiftright128(d[7], d[8], n);
#endif
	d[NB64BLOCK - 1] = ((int64_t)d[NB64BLOCK - 1]) >> n;

}

static void inline shiftL(unsigned char n, uint64_t* d) {

#if NB64BLOCK > 5
	d[8] = __shiftleft128(d[7], d[8], n);
	d[7] = __shiftleft128(d[6], d[7], n);
	d[6] = __shiftleft128(d[5], d[6], n);
	d[5] = __shiftleft128(d[4], d[5], n);
#endif
	d[4] = __shiftleft128(d[3], d[4], n);
	d[3] = __shiftleft128(d[2], d[3], n);
	d[2] = __shiftleft128(d[1], d[2], n);
	d[1] = __shiftleft128(d[0], d[1], n);
	d[0] = d[0] << n;

}

#endif // BIGINTH
/*
 * This file is part of the BSGS distribution (https://github.com/JeanLucPons/Kangaroo).
 * Copyright (c) 2020 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#include "IntGroup.h"

using namespace std;

IntGroup::IntGroup(int size) {
	this->size = size;
	subp = (Int*)malloc(size * sizeof(Int));
}

IntGroup::~IntGroup() {
	free(subp);
}

void IntGroup::Set(Int* pts) {
	ints = pts;
}

// Compute modular inversion of the whole group
void IntGroup::ModInv() {

	Int newValue;
	Int inverse;

	subp[0].Set(&ints[0]);
	for (int i = 1; i < size; i++) {
		subp[i].ModMulK1(&subp[i - 1], &ints[i]);
	}

	// Do the inversion
	inverse.Set(&subp[size - 1]);
	inverse.ModInv();

	for (int i = size - 1; i > 0; i--) {
		newValue.ModMulK1(&subp[i - 1], &inverse);
		inverse.ModMulK1(&ints[i]);
		ints[i].Set(&newValue);
	}

	ints[0].Set(&inverse);

}
/*
 * This file is part of the BSGS distribution (https://github.com/JeanLucPons/Kangaroo).
 * Copyright (c) 2020 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#ifndef INTGROUPH
#define INTGROUPH

#include "Int.h"
#include <vector>

class IntGroup {

public:

	IntGroup(int size);
	~IntGroup();
	void Set(Int* pts);
	void ModInv();

private:

	Int* ints;
	Int* subp;
	int size;

};

#endif // INTGROUPCPUH
/*
 * This file is part of the BSGS distribution (https://github.com/JeanLucPons/Kangaroo).
 * Copyright (c) 2020 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#include "Int.h"
#include <emmintrin.h>
#include <string.h>

#define MAX(x,y) (((x)>(y))?(x):(y))
#define MIN(x,y) (((x)<(y))?(x):(y))

static Int     _P;       // Field characteristic
static Int     _R;       // Montgomery multiplication R
static Int     _R2;      // Montgomery multiplication R2
static Int     _R3;      // Montgomery multiplication R3
static Int     _R4;      // Montgomery multiplication R4
static int32_t  Msize;    // Montgomery mult size
static uint32_t MM32;     // 32bits lsb negative inverse of P
static uint64_t MM64;     // 64bits lsb negative inverse of P
#define MSK62  0x3FFFFFFFFFFFFFFF

extern Int _ONE;

// ------------------------------------------------

void Int::ModAdd(Int* a) {
	Int p;
	Add(a);
	p.Sub(this, &_P);
	if (p.IsPositive())
		Set(&p);
}

// ------------------------------------------------

void Int::ModAdd(Int* a, Int* b) {
	Int p;
	Add(a, b);
	p.Sub(this, &_P);
	if (p.IsPositive())
		Set(&p);
}

// ------------------------------------------------

void Int::ModDouble() {
	Int p;
	Add(this);
	p.Sub(this, &_P);
	if (p.IsPositive())
		Set(&p);
}

// ------------------------------------------------

void Int::ModAdd(uint64_t a) {
	Int p;
	Add(a);
	p.Sub(this, &_P);
	if (p.IsPositive())
		Set(&p);
}

// ------------------------------------------------

void Int::ModSub(Int* a) {
	Sub(a);
	if (IsNegative())
		Add(&_P);
}

// ------------------------------------------------

void Int::ModSub(uint64_t a) {
	Sub(a);
	if (IsNegative())
		Add(&_P);
}

// ------------------------------------------------

void Int::ModSub(Int* a, Int* b) {
	Sub(a, b);
	if (IsNegative())
		Add(&_P);
}

// ------------------------------------------------

void Int::ModNeg() {
	Neg();
	Add(&_P);
}

// ------------------------------------------------

void Int::ModInv() {

	// Compute modular inverse of this mop _P
	// 0 < this < P  , P must be odd
	// Return 0 if no inverse

	// 256bit 
	//#define XCD 1               // ~62  kOps/s
	//#define BXCD 1              // ~167 kOps/s
	//#define MONTGOMERY 1        // ~200 kOps/s
	//#define PENK 1              // ~179 kOps/s
#define DRS62 1             // ~365 kOps/s

	Int u(&_P);
	Int v(this);
	Int r((int64_t)0);
	Int s((int64_t)1);

#ifdef XCD

	Int q, t1, t2, w;

	// Classic XCD 

	bool bIterations = true;  // Remember odd/even iterations
	while (!u.IsZero()) {
		// Step X3. Divide and "Subtract"
		q.Set(&v);
		q.Div(&u, &t2);   // q = u / v, t2 = u % v
		w.Mult(&q, &r);   // w = q * r
		t1.Add(&s, &w);   // t1 = s + w

						  // Swap u,v & r,s
		s.Set(&r);
		r.Set(&t1);
		v.Set(&u);
		u.Set(&t2);

		bIterations = !bIterations;
	}

	if (!v.IsOne()) {
		CLEAR();
		return;
	}

	if (!bIterations) {
		Set(&_P);
		Sub(&s);  /* inv = n - u1 */
	}
	else {
		Set(&s);  /* inv = u1     */
	}

#endif

#ifdef BXCD

#define SWAP_SUB(x,y) x.Sub(&y);y.Add(&x)

	// Binary XCD loop

	while (!u.IsZero()) {

		if (u.IsEven()) {

			u.ShiftR(1);
			if (r.IsOdd())
				r.Add(&_P);
			r.ShiftR(1);

		}
		else {

			SWAP_SUB(u, v);
			SWAP_SUB(r, s);

		}

	}

	// v ends with -1 ou 1
	if (!v.IsOne()) {
		// v = -1
		s.Neg();
		s.Add(&_P);
		v.Neg();
	}

	if (!v.IsOne()) {
		CLEAR();
		return;
	}

	if (s.IsNegative())
		s.Add(&_P);

	if (s.IsGreaterOrEqual(&_P))
		s.Sub(&_P);

	Set(&s);

#endif

#ifdef PENK

	Int x;
	Int n2(&_P);
	int k = 0;
	int T;
	int Q = _P.bits[0] & 3;
	shiftL(1, n2.bits64);

	// Penk's Algorithm (With DRS2 optimisation)

	while (v.IsEven()) {

		shiftR(1, v.bits64);
		if (s.IsEven())
			shiftR(1, s.bits64);
		else if (s.IsGreater(&_P)) {
			s.Sub(&_P);
			shiftR(1, s.bits64);
		}
		else {
			s.Add(&_P);
			shiftR(1, s.bits64);
		}

	}

	while (true) {

		if (u.IsGreater(&v)) {

			if ((u.bits[0] & 2) == (v.bits[0] & 2)) {
				u.Sub(&v);
				r.Sub(&s);
			}
			else {
				u.Add(&v);
				r.Add(&s);
			}
			shiftR(2, u.bits64);
			T = r.bits[0] & 3;
			if (T == 0) {
				shiftR(2, r.bits64);
			}
			else if (T == 2) {
				r.Add(&n2);
				shiftR(2, r.bits64);
			}
			else if (T == Q) {
				r.Sub(&_P);
				shiftR(2, r.bits64);
			}
			else {
				r.Add(&_P);
				shiftR(2, r.bits64);
			}
			while (u.IsEven()) {
				shiftR(1, u.bits64);
				if (r.IsEven()) {
					shiftR(1, r.bits64);
				}
				else if (r.IsGreater(&_P)) {
					r.Sub(&_P);
					shiftR(1, r.bits64);
				}
				else {
					r.Add(&_P);
					shiftR(1, r.bits64);
				}
			}

		}
		else {

			if ((u.bits[0] & 2) == (v.bits[0] & 2)) {
				v.Sub(&u);
				s.Sub(&r);
			}
			else {
				v.Add(&u);
				s.Add(&r);
			}

			if (v.IsZero())
				break;

			shiftR(2, v.bits64);
			T = s.bits[0] & 3;
			if (T == 0) {
				shiftR(2, s.bits64);
			}
			else if (T == 2) {
				s.Add(&n2);
				shiftR(2, s.bits64);
			}
			else if (T == Q) {
				s.Sub(&_P);
				shiftR(2, s.bits64);
			}
			else {
				s.Add(&_P);
				shiftR(2, s.bits64);
			}

			while (v.IsEven()) {
				shiftR(1, v.bits64);
				if (s.IsEven()) {
					shiftR(1, s.bits64);
				}
				else if (s.IsGreater(&_P)) {
					s.Sub(&_P);
					shiftR(1, s.bits64);
				}
				else {
					s.Add(&_P);
					shiftR(1, s.bits64);
				}
			}

		}

	}

	if (u.IsGreater(&_ONE)) {
		CLEAR();
		return;
	}
	if (r.IsNegative())
		r.Add(&_P);
	Set(&r);

#endif

#ifdef MONTGOMERY

	Int x;
	int k = 0;

	// Montgomery method
	while (v.IsStrictPositive()) {
		if (u.IsEven()) {
			shiftR(1, u.bits64);
			shiftL(1, s.bits64);
		}
		else if (v.IsEven()) {
			shiftR(1, v.bits64);
			shiftL(1, r.bits64);
		}
		else {
			x.Set(&u);
			x.Sub(&v);
			if (x.IsStrictPositive()) {
				shiftR(1, x.bits64);
				u.Set(&x);
				r.Add(&s);
				shiftL(1, s.bits64);
			}
			else {
				x.Neg();
				shiftR(1, x.bits64);
				v.Set(&x);
				s.Add(&r);
				shiftL(1, r.bits64);
			}
		}
		k++;
	}

	if (r.IsGreater(&_P))
		r.Sub(&_P);
	r.Neg();
	r.Add(&_P);

	for (int i = 0; i < k; i++) {
		if (r.IsEven()) {
			shiftR(1, r.bits64);
		}
		else {
			r.Add(&_P);
			shiftR(1, r.bits64);
		}
	}
	Set(&r);

#endif

#ifdef DRS62

	// Delayed right shift 62bits

#define SWAP_ADD(x,y) x+=y;y-=x;
#define SWAP_SUB(x,y) x-=y;y+=x;
#define IS_EVEN(x) ((x&1)==0)

	Int r0_P;
	Int s0_P;
	Int uu_u;
	Int uv_v;
	Int vu_u;
	Int vv_v;
	Int uu_r;
	Int uv_s;
	Int vu_r;
	Int vv_s;

	int64_t bitCount;
	int64_t uu, uv, vu, vv;
	int64_t v0, u0;
	int64_t nb0;

	while (!u.IsZero()) {

		// u' = (uu*u + uv*v) >> bitCount
		// v' = (vu*u + vv*v) >> bitCount
		// Do not maintain a matrix for r and s, the number of 
		// 'added P' can be easily calculated
		uu = 1; uv = 0;
		vu = 0; vv = 1;

		u0 = (int64_t)u.bits64[0];
		v0 = (int64_t)v.bits64[0];
		bitCount = 0;

		// Slightly optimized Binary XCD loop on native signed integers
		// Stop at 62 bits to avoid uv matrix overfow and loss of sign bit
		while (true) {

			while (IS_EVEN(u0) && bitCount < 62) {

				bitCount++;
				u0 >>= 1;
				vu <<= 1;
				vv <<= 1;

			}

			if (bitCount == 62)
				break;

			nb0 = (v0 + u0) & 0x3;
			if (nb0 == 0) {
				SWAP_ADD(uv, vv);
				SWAP_ADD(uu, vu);
				SWAP_ADD(u0, v0);
			}
			else {
				SWAP_SUB(uv, vv);
				SWAP_SUB(uu, vu);
				SWAP_SUB(u0, v0);
			}

		}

		// Now update BigInt variables

		uu_u.IMult(&u, uu);
		uv_v.IMult(&v, uv);

		vu_u.IMult(&u, vu);
		vv_v.IMult(&v, vv);

		uu_r.IMult(&r, uu);
		uv_s.IMult(&s, uv);

		vu_r.IMult(&r, vu);
		vv_s.IMult(&s, vv);

		// Compute multiple of P to add to s and r to make them multiple of 2^62
		uint64_t r0 = ((uu_r.bits64[0] + uv_s.bits64[0]) * MM64) & MSK62;
		uint64_t s0 = ((vu_r.bits64[0] + vv_s.bits64[0]) * MM64) & MSK62;
		r0_P.Mult(&_P, r0);
		s0_P.Mult(&_P, s0);

		// u = (uu*u + uv*v)
		u.Add(&uu_u, &uv_v);

		// v = (vu*u + vv*v)
		v.Add(&vu_u, &vv_v);

		// r = (uu*r + uv*s + r0*P)
		r.Add(&uu_r, &uv_s);
		r.Add(&r0_P);

		// s = (vu*r + vv*s + s0*P)
		s.Add(&vu_r, &vv_s);
		s.Add(&s0_P);

		// Right shift all variables by 62bits
		shiftR(62, u.bits64);
		shiftR(62, v.bits64);
		shiftR(62, r.bits64);
		shiftR(62, s.bits64);

	}

	// v ends with -1 or 1
	if (v.IsNegative()) {
		// V = -1
		v.Neg();
		s.Neg();
		s.Add(&_P);
	}
	if (!v.IsOne()) {
		// No inverse
		CLEAR();
		return;
	}

	if (s.IsNegative())
		s.Add(&_P);

	if (s.IsGreaterOrEqual(&_P))
		s.Sub(&_P);

	Set(&s);

#endif

}

// ------------------------------------------------

void Int::ModExp(Int* e) {

	Int base(this);
	SetInt32(1);
	uint32_t i = 0;

	uint32_t nbBit = e->GetBitLength();
	for (int i = 0; i < (int)nbBit; i++) {
		if (e->GetBit(i))
			ModMul(&base);
		base.ModMul(&base);
	}

}

// ------------------------------------------------

void Int::ModMul(Int* a) {

	Int p;
	p.MontgomeryMult(a, this);
	MontgomeryMult(&_R2, &p);

}

// ------------------------------------------------

void Int::ModSquare(Int* a) {

	Int p;
	p.MontgomeryMult(a, a);
	MontgomeryMult(&_R2, &p);

}

// ------------------------------------------------

void Int::ModCube(Int* a) {

	Int p;
	Int p2;
	p.MontgomeryMult(a, a);
	p2.MontgomeryMult(&p, a);
	MontgomeryMult(&_R3, &p2);

}

// ------------------------------------------------

bool Int::HasSqrt() {

	// Euler's criterion
	Int e(&_P);
	Int a(this);
	e.SubOne();
	e.ShiftR(1);
	a.ModExp(&e);

	return a.IsOne();

}

// ------------------------------------------------

void Int::ModSqrt() {

	if (_P.IsEven()) {
		CLEAR();
		return;
	}

	if (!HasSqrt()) {
		CLEAR();
		return;
	}

	if ((_P.bits64[0] & 3) == 3) {

		Int e(&_P);
		e.AddOne();
		e.ShiftR(2);
		ModExp(&e);

	}
	else if ((_P.bits64[0] & 3) == 1) {

		int nbBit = _P.GetBitLength();

		// Tonelli Shanks
		uint64_t e = 0;
		Int S(&_P);
		S.SubOne();
		while (S.IsEven()) {
			S.ShiftR(1);
			e++;
		}

		// Search smalest non-qresidue of P
		Int q((uint64_t)1);
		do {
			q.AddOne();
		} while (q.HasSqrt());

		Int c(&q);
		c.ModExp(&S);

		Int t(this);
		t.ModExp(&S);

		Int r(this);
		Int ex(&S);
		ex.AddOne();
		ex.ShiftR(1);
		r.ModExp(&ex);

		uint64_t M = e;
		while (!t.IsOne()) {

			Int t2(&t);
			uint64_t i = 0;
			while (!t2.IsOne()) {
				t2.ModSquare(&t2);
				i++;
			}

			Int b(&c);
			for (uint64_t j = 0; j < M - i - 1; j++)
				b.ModSquare(&b);
			M = i;
			c.ModSquare(&b);
			t.ModMul(&t, &c);
			r.ModMul(&r, &b);

		}

		Set(&r);

	}

}

// ------------------------------------------------

void Int::ModMul(Int* a, Int* b) {

	Int p;
	p.MontgomeryMult(a, b);
	MontgomeryMult(&_R2, &p);

}

// ------------------------------------------------

Int* Int::GetFieldCharacteristic() {
	return &_P;
}

// ------------------------------------------------

Int* Int::GetR() {
	return &_R;
}
Int* Int::GetR2() {
	return &_R2;
}
Int* Int::GetR3() {
	return &_R3;
}
Int* Int::GetR4() {
	return &_R4;
}

// ------------------------------------------------

void Int::SetupField(Int* n, Int* R, Int* R2, Int* R3, Int* R4) {

	// Size in number of 32bit word
	int nSize = n->GetSize();

	// Last digit inversions (Newton's iteration)
	{
		int64_t x, t;
		x = t = (int64_t)n->bits64[0];
		x = x * (2 - t * x);
		x = x * (2 - t * x);
		x = x * (2 - t * x);
		x = x * (2 - t * x);
		x = x * (2 - t * x);
		MM64 = (uint64_t)(-x);
		MM32 = (uint32_t)MM64;
	}
	_P.Set(n);

	// Size of Montgomery mult (64bits digit)
	Msize = nSize / 2;

	// Compute few power of R
	// R = 2^(64*Msize) mod n
	Int Ri;
	Ri.MontgomeryMult(&_ONE, &_ONE); // Ri = R^-1
	_R.Set(&Ri);                     // R  = R^-1
	_R2.MontgomeryMult(&Ri, &_ONE);  // R2 = R^-2
	_R3.MontgomeryMult(&Ri, &Ri);    // R3 = R^-3
	_R4.MontgomeryMult(&_R3, &_ONE); // R4 = R^-4

	_R.ModInv();                     // R  = R
	_R2.ModInv();                    // R2 = R^2
	_R3.ModInv();                    // R3 = R^3
	_R4.ModInv();                    // R4 = R^4

	if (R)
		R->Set(&_R);

	if (R2)
		R2->Set(&_R2);

	if (R3)
		R3->Set(&_R3);

	if (R4)
		R4->Set(&_R4);

}
// ------------------------------------------------

uint64_t Int::AddC(Int* a) {

	unsigned char c = 0;
	c = _addcarry_u64(c, bits64[0], a->bits64[0], bits64 + 0);
	c = _addcarry_u64(c, bits64[1], a->bits64[1], bits64 + 1);
	c = _addcarry_u64(c, bits64[2], a->bits64[2], bits64 + 2);
	c = _addcarry_u64(c, bits64[3], a->bits64[3], bits64 + 3);
	c = _addcarry_u64(c, bits64[4], a->bits64[4], bits64 + 4);
#if NB64BLOCK > 5
	c = _addcarry_u64(c, bits64[5], a->bits64[5], bits64 + 5);
	c = _addcarry_u64(c, bits64[6], a->bits64[6], bits64 + 6);
	c = _addcarry_u64(c, bits64[7], a->bits64[7], bits64 + 7);
	c = _addcarry_u64(c, bits64[8], a->bits64[8], bits64 + 8);
#endif

	return c;

}

// ------------------------------------------------

void Int::AddAndShift(Int* a, Int* b, uint64_t cH) {

	unsigned char c = 0;
	c = _addcarry_u64(c, b->bits64[0], a->bits64[0], bits64 + 0);
	c = _addcarry_u64(c, b->bits64[1], a->bits64[1], bits64 + 0);
	c = _addcarry_u64(c, b->bits64[2], a->bits64[2], bits64 + 1);
	c = _addcarry_u64(c, b->bits64[3], a->bits64[3], bits64 + 2);
	c = _addcarry_u64(c, b->bits64[4], a->bits64[4], bits64 + 3);
#if NB64BLOCK > 5
	c = _addcarry_u64(c, b->bits64[5], a->bits64[5], bits64 + 4);
	c = _addcarry_u64(c, b->bits64[6], a->bits64[6], bits64 + 5);
	c = _addcarry_u64(c, b->bits64[7], a->bits64[7], bits64 + 6);
	c = _addcarry_u64(c, b->bits64[8], a->bits64[8], bits64 + 7);
#endif

	bits64[NB64BLOCK - 1] = c + cH;

}

// ------------------------------------------------
void Int::MontgomeryMult(Int* a) {

	// Compute a*b*R^-1 (mod n),  R=2^k (mod n), k = Msize*64
	// a and b must be lower than n
	// See SetupField()

	Int t;
	Int pr;
	Int p;
	uint64_t ML;
	uint64_t c;

	// i = 0
	imm_umul(a->bits64, bits64[0], pr.bits64);
	ML = pr.bits64[0] * MM64;
	imm_umul(_P.bits64, ML, p.bits64);
	c = pr.AddC(&p);
	memcpy(t.bits64, pr.bits64 + 1, 8 * (NB64BLOCK - 1));
	t.bits64[NB64BLOCK - 1] = c;

	for (int i = 1; i < Msize; i++) {

		imm_umul(a->bits64, bits64[i], pr.bits64);
		ML = (pr.bits64[0] + t.bits64[0]) * MM64;
		imm_umul(_P.bits64, ML, p.bits64);
		c = pr.AddC(&p);
		t.AddAndShift(&t, &pr, c);

	}

	p.Sub(&t, &_P);
	if (p.IsPositive())
		Set(&p);
	else
		Set(&t);

}

void Int::MontgomeryMult(Int* a, Int* b) {

	// Compute a*b*R^-1 (mod n),  R=2^k (mod n), k = Msize*64
	// a and b must be lower than n
	// See SetupField()

	Int pr;
	Int p;
	uint64_t ML;
	uint64_t c;

	// i = 0
	imm_umul(a->bits64, b->bits64[0], pr.bits64);
	ML = pr.bits64[0] * MM64;
	imm_umul(_P.bits64, ML, p.bits64);
	c = pr.AddC(&p);
	memcpy(bits64, pr.bits64 + 1, 8 * (NB64BLOCK - 1));
	bits64[NB64BLOCK - 1] = c;

	for (int i = 1; i < Msize; i++) {

		imm_umul(a->bits64, b->bits64[i], pr.bits64);
		ML = (pr.bits64[0] + bits64[0]) * MM64;
		imm_umul(_P.bits64, ML, p.bits64);
		c = pr.AddC(&p);
		AddAndShift(this, &pr, c);

	}

	p.Sub(this, &_P);
	if (p.IsPositive())
		Set(&p);

}


// SecpK1 specific section -----------------------------------------------------------------------------

void Int::ModMulK1(Int* a, Int* b) {

#ifndef WIN64
#if (__GNUC__ > 7) || (__GNUC__ == 7 && (__GNUC_MINOR__ > 2))
	unsigned char c;
#else
	#warning "GCC lass than 7.3 detected, upgrade gcc to get best perfromance"
		volatile unsigned char c;
#endif
#else
	unsigned char c;
#endif


	uint64_t ah, al;
	uint64_t t[5];
	uint64_t r512[8];
	r512[5] = 0;
	r512[6] = 0;
	r512[7] = 0;

	// 256*256 multiplier
	imm_umul(a->bits64, b->bits64[0], r512);
	imm_umul(a->bits64, b->bits64[1], t);
	c = _addcarry_u64(0, r512[1], t[0], r512 + 1);
	c = _addcarry_u64(c, r512[2], t[1], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[2], r512 + 3);
	c = _addcarry_u64(c, r512[4], t[3], r512 + 4);
	c = _addcarry_u64(c, r512[5], t[4], r512 + 5);
	imm_umul(a->bits64, b->bits64[2], t);
	c = _addcarry_u64(0, r512[2], t[0], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[1], r512 + 3);
	c = _addcarry_u64(c, r512[4], t[2], r512 + 4);
	c = _addcarry_u64(c, r512[5], t[3], r512 + 5);
	c = _addcarry_u64(c, r512[6], t[4], r512 + 6);
	imm_umul(a->bits64, b->bits64[3], t);
	c = _addcarry_u64(0, r512[3], t[0], r512 + 3);
	c = _addcarry_u64(c, r512[4], t[1], r512 + 4);
	c = _addcarry_u64(c, r512[5], t[2], r512 + 5);
	c = _addcarry_u64(c, r512[6], t[3], r512 + 6);
	c = _addcarry_u64(c, r512[7], t[4], r512 + 7);

	// Reduce from 512 to 320 
	imm_umul(r512 + 4, 0x1000003D1ULL, t);
	c = _addcarry_u64(0, r512[0], t[0], r512 + 0);
	c = _addcarry_u64(c, r512[1], t[1], r512 + 1);
	c = _addcarry_u64(c, r512[2], t[2], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[3], r512 + 3);

	// Reduce from 320 to 256 
	// No overflow possible here t[4]+c<=0x1000003D1ULL
	al = _umul128(t[4] + c, 0x1000003D1ULL, &ah);
	c = _addcarry_u64(0, r512[0], al, bits64 + 0);
	c = _addcarry_u64(c, r512[1], ah, bits64 + 1);
	c = _addcarry_u64(c, r512[2], 0ULL, bits64 + 2);
	c = _addcarry_u64(c, r512[3], 0ULL, bits64 + 3);

	// Probability of carry here or that this>P is very very unlikely
	bits64[4] = 0;

}

void Int::ModMulK1(Int* a) {

#ifndef WIN64
#if (__GNUC__ > 7) || (__GNUC__ == 7 && (__GNUC_MINOR__ > 2))
	unsigned char c;
#else
	#warning "GCC lass than 7.3 detected, upgrade gcc to get best perfromance"
		volatile unsigned char c;
#endif
#else
	unsigned char c;
#endif

	uint64_t ah, al;
	uint64_t t[5];
	uint64_t r512[8];
	r512[5] = 0;
	r512[6] = 0;
	r512[7] = 0;

	// 256*256 multiplier
	imm_umul(a->bits64, bits64[0], r512);
	imm_umul(a->bits64, bits64[1], t);
	c = _addcarry_u64(0, r512[1], t[0], r512 + 1);
	c = _addcarry_u64(c, r512[2], t[1], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[2], r512 + 3);
	c = _addcarry_u64(c, r512[4], t[3], r512 + 4);
	c = _addcarry_u64(c, r512[5], t[4], r512 + 5);
	imm_umul(a->bits64, bits64[2], t);
	c = _addcarry_u64(0, r512[2], t[0], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[1], r512 + 3);
	c = _addcarry_u64(c, r512[4], t[2], r512 + 4);
	c = _addcarry_u64(c, r512[5], t[3], r512 + 5);
	c = _addcarry_u64(c, r512[6], t[4], r512 + 6);
	imm_umul(a->bits64, bits64[3], t);
	c = _addcarry_u64(0, r512[3], t[0], r512 + 3);
	c = _addcarry_u64(c, r512[4], t[1], r512 + 4);
	c = _addcarry_u64(c, r512[5], t[2], r512 + 5);
	c = _addcarry_u64(c, r512[6], t[3], r512 + 6);
	c = _addcarry_u64(c, r512[7], t[4], r512 + 7);

	// Reduce from 512 to 320 
	imm_umul(r512 + 4, 0x1000003D1ULL, t);
	c = _addcarry_u64(0, r512[0], t[0], r512 + 0);
	c = _addcarry_u64(c, r512[1], t[1], r512 + 1);
	c = _addcarry_u64(c, r512[2], t[2], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[3], r512 + 3);

	// Reduce from 320 to 256 
	// No overflow possible here t[4]+c<=0x1000003D1ULL
	al = _umul128(t[4] + c, 0x1000003D1ULL, &ah);
	c = _addcarry_u64(0, r512[0], al, bits64 + 0);
	c = _addcarry_u64(c, r512[1], ah, bits64 + 1);
	c = _addcarry_u64(c, r512[2], 0, bits64 + 2);
	c = _addcarry_u64(c, r512[3], 0, bits64 + 3);
	// Probability of carry here or that this>P is very very unlikely
	bits64[4] = 0;

}

void Int::ModSquareK1(Int* a) {

#ifndef WIN64
#if (__GNUC__ > 7) || (__GNUC__ == 7 && (__GNUC_MINOR__ > 2))
	unsigned char c;
#else
	#warning "GCC lass than 7.3 detected, upgrade gcc to get best perfromance"
		volatile unsigned char c;
#endif
#else
	unsigned char c;
#endif

	uint64_t r512[8];
	uint64_t u10, u11;
	uint64_t t1;
	uint64_t t2;
	uint64_t t[5];


	//k=0
	r512[0] = _umul128(a->bits64[0], a->bits64[0], &t[1]);

	//k=1
	t[3] = _umul128(a->bits64[0], a->bits64[1], &t[4]);
	c = _addcarry_u64(0, t[3], t[3], &t[3]);
	c = _addcarry_u64(c, t[4], t[4], &t[4]);
	c = _addcarry_u64(c, 0, 0, &t1);
	c = _addcarry_u64(0, t[1], t[3], &t[3]);
	c = _addcarry_u64(c, t[4], 0, &t[4]);
	c = _addcarry_u64(c, t1, 0, &t1);
	r512[1] = t[3];

	//k=2
	t[0] = _umul128(a->bits64[0], a->bits64[2], &t[1]);
	c = _addcarry_u64(0, t[0], t[0], &t[0]);
	c = _addcarry_u64(c, t[1], t[1], &t[1]);
	c = _addcarry_u64(c, 0, 0, &t2);

	u10 = _umul128(a->bits64[1], a->bits64[1], &u11);
	c = _addcarry_u64(0, t[0], u10, &t[0]);
	c = _addcarry_u64(c, t[1], u11, &t[1]);
	c = _addcarry_u64(c, t2, 0, &t2);
	c = _addcarry_u64(0, t[0], t[4], &t[0]);
	c = _addcarry_u64(c, t[1], t1, &t[1]);
	c = _addcarry_u64(c, t2, 0, &t2);
	r512[2] = t[0];

	//k=3
	t[3] = _umul128(a->bits64[0], a->bits64[3], &t[4]);
	u10 = _umul128(a->bits64[1], a->bits64[2], &u11);

	c = _addcarry_u64(0, t[3], u10, &t[3]);
	c = _addcarry_u64(c, t[4], u11, &t[4]);
	c = _addcarry_u64(c, 0, 0, &t1);
	t1 += t1;
	c = _addcarry_u64(0, t[3], t[3], &t[3]);
	c = _addcarry_u64(c, t[4], t[4], &t[4]);
	c = _addcarry_u64(c, t1, 0, &t1);
	c = _addcarry_u64(0, t[3], t[1], &t[3]);
	c = _addcarry_u64(c, t[4], t2, &t[4]);
	c = _addcarry_u64(c, t1, 0, &t1);
	r512[3] = t[3];

	//k=4
	t[0] = _umul128(a->bits64[1], a->bits64[3], &t[1]);
	c = _addcarry_u64(0, t[0], t[0], &t[0]);
	c = _addcarry_u64(c, t[1], t[1], &t[1]);
	c = _addcarry_u64(c, 0, 0, &t2);

	u10 = _umul128(a->bits64[2], a->bits64[2], &u11);
	c = _addcarry_u64(0, t[0], u10, &t[0]);
	c = _addcarry_u64(c, t[1], u11, &t[1]);
	c = _addcarry_u64(c, t2, 0, &t2);
	c = _addcarry_u64(0, t[0], t[4], &t[0]);
	c = _addcarry_u64(c, t[1], t1, &t[1]);
	c = _addcarry_u64(c, t2, 0, &t2);
	r512[4] = t[0];

	//k=5
	t[3] = _umul128(a->bits64[2], a->bits64[3], &t[4]);
	c = _addcarry_u64(0, t[3], t[3], &t[3]);
	c = _addcarry_u64(c, t[4], t[4], &t[4]);
	c = _addcarry_u64(c, 0, 0, &t1);
	c = _addcarry_u64(0, t[3], t[1], &t[3]);
	c = _addcarry_u64(c, t[4], t2, &t[4]);
	c = _addcarry_u64(c, t1, 0, &t1);
	r512[5] = t[3];

	//k=6
	t[0] = _umul128(a->bits64[3], a->bits64[3], &t[1]);
	c = _addcarry_u64(0, t[0], t[4], &t[0]);
	c = _addcarry_u64(c, t[1], t1, &t[1]);
	r512[6] = t[0];

	//k=7
	r512[7] = t[1];

	// Reduce from 512 to 320 
	// Reduce from 512 to 320 
	imm_umul(r512 + 4, 0x1000003D1ULL, t);
	c = _addcarry_u64(0, r512[0], t[0], r512 + 0);
	c = _addcarry_u64(c, r512[1], t[1], r512 + 1);
	c = _addcarry_u64(c, r512[2], t[2], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[3], r512 + 3);

	// Reduce from 320 to 256 
	// No overflow possible here t[4]+c<=0x1000003D1ULL
	u10 = _umul128(t[4] + c, 0x1000003D1ULL, &u11);
	c = _addcarry_u64(0, r512[0], u10, bits64 + 0);
	c = _addcarry_u64(c, r512[1], u11, bits64 + 1);
	c = _addcarry_u64(c, r512[2], 0, bits64 + 2);
	c = _addcarry_u64(c, r512[3], 0, bits64 + 3);
	// Probability of carry here or that this>P is very very unlikely
	bits64[4] = 0;

}

static Int _R2o;                               // R^2 for SecpK1 order modular mult
static uint64_t MM64o = 0x4B0DFF665588B13FULL; // 64bits lsb negative inverse of SecpK1 order
static Int* _O;                                // SecpK1 order

void Int::InitK1(Int* order) {
	_O = order;
	_R2o.SetBase16("9D671CD581C69BC5E697F5E45BCD07C6741496C20E7CF878896CF21467D7D140");
}

void Int::ModAddK1order(Int* a, Int* b) {
	Add(a, b);
	Sub(_O);
	if (IsNegative())
		Add(_O);
}

void Int::ModAddK1order(Int* a) {
	Add(a);
	Sub(_O);
	if (IsNegative())
		Add(_O);
}

void Int::ModSubK1order(Int* a) {
	Sub(a);
	if (IsNegative())
		Add(_O);
}

void Int::ModNegK1order() {
	Neg();
	Add(_O);
}

uint32_t Int::ModPositiveK1() {

	Int N(this);
	Int D(this);
	N.ModNeg();
	D.Sub(&N);
	if (D.IsNegative()) {
		return 0;
	}
	else {
		Set(&N);
		return 1;
	}

}


void Int::ModMulK1order(Int* a) {

	Int t;
	Int pr;
	Int p;
	uint64_t ML;
	uint64_t c;

	imm_umul(a->bits64, bits64[0], pr.bits64);
	ML = pr.bits64[0] * MM64o;
	imm_umul(_O->bits64, ML, p.bits64);
	c = pr.AddC(&p);
	memcpy(t.bits64, pr.bits64 + 1, 32);
	t.bits64[4] = c;

	for (int i = 1; i < 4; i++) {

		imm_umul(a->bits64, bits64[i], pr.bits64);
		ML = (pr.bits64[0] + t.bits64[0]) * MM64o;
		imm_umul(_O->bits64, ML, p.bits64);
		c = pr.AddC(&p);
		t.AddAndShift(&t, &pr, c);

	}

	p.Sub(&t, _O);
	if (p.IsPositive())
		Set(&p);
	else
		Set(&t);


	// Normalize

	imm_umul(_R2o.bits64, bits64[0], pr.bits64);
	ML = pr.bits64[0] * MM64o;
	imm_umul(_O->bits64, ML, p.bits64);
	c = pr.AddC(&p);
	memcpy(t.bits64, pr.bits64 + 1, 32);
	t.bits64[4] = c;

	for (int i = 1; i < 4; i++) {

		imm_umul(_R2o.bits64, bits64[i], pr.bits64);
		ML = (pr.bits64[0] + t.bits64[0]) * MM64o;
		imm_umul(_O->bits64, ML, p.bits64);
		c = pr.AddC(&p);
		t.AddAndShift(&t, &pr, c);

	}

	p.Sub(&t, _O);
	if (p.IsPositive())
		Set(&p);
	else
		Set(&t);

}
/*
 * This file is part of the BSGS distribution (https://github.com/JeanLucPons/Kangaroo).
 * Copyright (c) 2020 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#include "Int.h"
#include <emmintrin.h>
#include <string.h>

#define MAX(x,y) (((x)>(y))?(x):(y))
#define MIN(x,y) (((x)<(y))?(x):(y))

static Int     _P;       // Field characteristic
static Int     _R;       // Montgomery multiplication R
static Int     _R2;      // Montgomery multiplication R2
static Int     _R3;      // Montgomery multiplication R3
static Int     _R4;      // Montgomery multiplication R4
static int32_t  Msize;    // Montgomery mult size
static uint32_t MM32;     // 32bits lsb negative inverse of P
static uint64_t MM64;     // 64bits lsb negative inverse of P
#define MSK62  0x3FFFFFFFFFFFFFFF

extern Int _ONE;

// ------------------------------------------------

void Int::ModAdd(Int* a) {
	Int p;
	Add(a);
	p.Sub(this, &_P);
	if (p.IsPositive())
		Set(&p);
}

// ------------------------------------------------

void Int::ModAdd(Int* a, Int* b) {
	Int p;
	Add(a, b);
	p.Sub(this, &_P);
	if (p.IsPositive())
		Set(&p);
}

// ------------------------------------------------

void Int::ModDouble() {
	Int p;
	Add(this);
	p.Sub(this, &_P);
	if (p.IsPositive())
		Set(&p);
}

// ------------------------------------------------

void Int::ModAdd(uint64_t a) {
	Int p;
	Add(a);
	p.Sub(this, &_P);
	if (p.IsPositive())
		Set(&p);
}

// ------------------------------------------------

void Int::ModSub(Int* a) {
	Sub(a);
	if (IsNegative())
		Add(&_P);
}

// ------------------------------------------------

void Int::ModSub(uint64_t a) {
	Sub(a);
	if (IsNegative())
		Add(&_P);
}

// ------------------------------------------------

void Int::ModSub(Int* a, Int* b) {
	Sub(a, b);
	if (IsNegative())
		Add(&_P);
}

// ------------------------------------------------

void Int::ModNeg() {
	Neg();
	Add(&_P);
}

// ------------------------------------------------

void Int::ModInv() {

	// Compute modular inverse of this mop _P
	// 0 < this < P  , P must be odd
	// Return 0 if no inverse

	// 256bit 
	//#define XCD 1               // ~62  kOps/s
	//#define BXCD 1              // ~167 kOps/s
	//#define MONTGOMERY 1        // ~200 kOps/s
	//#define PENK 1              // ~179 kOps/s
#define DRS62 1             // ~365 kOps/s

	Int u(&_P);
	Int v(this);
	Int r((int64_t)0);
	Int s((int64_t)1);

#ifdef XCD

	Int q, t1, t2, w;

	// Classic XCD 

	bool bIterations = true;  // Remember odd/even iterations
	while (!u.IsZero()) {
		// Step X3. Divide and "Subtract"
		q.Set(&v);
		q.Div(&u, &t2);   // q = u / v, t2 = u % v
		w.Mult(&q, &r);   // w = q * r
		t1.Add(&s, &w);   // t1 = s + w

						  // Swap u,v & r,s
		s.Set(&r);
		r.Set(&t1);
		v.Set(&u);
		u.Set(&t2);

		bIterations = !bIterations;
	}

	if (!v.IsOne()) {
		CLEAR();
		return;
	}

	if (!bIterations) {
		Set(&_P);
		Sub(&s);  /* inv = n - u1 */
	}
	else {
		Set(&s);  /* inv = u1     */
	}

#endif

#ifdef BXCD

#define SWAP_SUB(x,y) x.Sub(&y);y.Add(&x)

	// Binary XCD loop

	while (!u.IsZero()) {

		if (u.IsEven()) {

			u.ShiftR(1);
			if (r.IsOdd())
				r.Add(&_P);
			r.ShiftR(1);

		}
		else {

			SWAP_SUB(u, v);
			SWAP_SUB(r, s);

		}

	}

	// v ends with -1 ou 1
	if (!v.IsOne()) {
		// v = -1
		s.Neg();
		s.Add(&_P);
		v.Neg();
	}

	if (!v.IsOne()) {
		CLEAR();
		return;
	}

	if (s.IsNegative())
		s.Add(&_P);

	if (s.IsGreaterOrEqual(&_P))
		s.Sub(&_P);

	Set(&s);

#endif

#ifdef PENK

	Int x;
	Int n2(&_P);
	int k = 0;
	int T;
	int Q = _P.bits[0] & 3;
	shiftL(1, n2.bits64);

	// Penk's Algorithm (With DRS2 optimisation)

	while (v.IsEven()) {

		shiftR(1, v.bits64);
		if (s.IsEven())
			shiftR(1, s.bits64);
		else if (s.IsGreater(&_P)) {
			s.Sub(&_P);
			shiftR(1, s.bits64);
		}
		else {
			s.Add(&_P);
			shiftR(1, s.bits64);
		}

	}

	while (true) {

		if (u.IsGreater(&v)) {

			if ((u.bits[0] & 2) == (v.bits[0] & 2)) {
				u.Sub(&v);
				r.Sub(&s);
			}
			else {
				u.Add(&v);
				r.Add(&s);
			}
			shiftR(2, u.bits64);
			T = r.bits[0] & 3;
			if (T == 0) {
				shiftR(2, r.bits64);
			}
			else if (T == 2) {
				r.Add(&n2);
				shiftR(2, r.bits64);
			}
			else if (T == Q) {
				r.Sub(&_P);
				shiftR(2, r.bits64);
			}
			else {
				r.Add(&_P);
				shiftR(2, r.bits64);
			}
			while (u.IsEven()) {
				shiftR(1, u.bits64);
				if (r.IsEven()) {
					shiftR(1, r.bits64);
				}
				else if (r.IsGreater(&_P)) {
					r.Sub(&_P);
					shiftR(1, r.bits64);
				}
				else {
					r.Add(&_P);
					shiftR(1, r.bits64);
				}
			}

		}
		else {

			if ((u.bits[0] & 2) == (v.bits[0] & 2)) {
				v.Sub(&u);
				s.Sub(&r);
			}
			else {
				v.Add(&u);
				s.Add(&r);
			}

			if (v.IsZero())
				break;

			shiftR(2, v.bits64);
			T = s.bits[0] & 3;
			if (T == 0) {
				shiftR(2, s.bits64);
			}
			else if (T == 2) {
				s.Add(&n2);
				shiftR(2, s.bits64);
			}
			else if (T == Q) {
				s.Sub(&_P);
				shiftR(2, s.bits64);
			}
			else {
				s.Add(&_P);
				shiftR(2, s.bits64);
			}

			while (v.IsEven()) {
				shiftR(1, v.bits64);
				if (s.IsEven()) {
					shiftR(1, s.bits64);
				}
				else if (s.IsGreater(&_P)) {
					s.Sub(&_P);
					shiftR(1, s.bits64);
				}
				else {
					s.Add(&_P);
					shiftR(1, s.bits64);
				}
			}

		}

	}

	if (u.IsGreater(&_ONE)) {
		CLEAR();
		return;
	}
	if (r.IsNegative())
		r.Add(&_P);
	Set(&r);

#endif

#ifdef MONTGOMERY

	Int x;
	int k = 0;

	// Montgomery method
	while (v.IsStrictPositive()) {
		if (u.IsEven()) {
			shiftR(1, u.bits64);
			shiftL(1, s.bits64);
		}
		else if (v.IsEven()) {
			shiftR(1, v.bits64);
			shiftL(1, r.bits64);
		}
		else {
			x.Set(&u);
			x.Sub(&v);
			if (x.IsStrictPositive()) {
				shiftR(1, x.bits64);
				u.Set(&x);
				r.Add(&s);
				shiftL(1, s.bits64);
			}
			else {
				x.Neg();
				shiftR(1, x.bits64);
				v.Set(&x);
				s.Add(&r);
				shiftL(1, r.bits64);
			}
		}
		k++;
	}

	if (r.IsGreater(&_P))
		r.Sub(&_P);
	r.Neg();
	r.Add(&_P);

	for (int i = 0; i < k; i++) {
		if (r.IsEven()) {
			shiftR(1, r.bits64);
		}
		else {
			r.Add(&_P);
			shiftR(1, r.bits64);
		}
	}
	Set(&r);

#endif

#ifdef DRS62

	// Delayed right shift 62bits

#define SWAP_ADD(x,y) x+=y;y-=x;
#define SWAP_SUB(x,y) x-=y;y+=x;
#define IS_EVEN(x) ((x&1)==0)

	Int r0_P;
	Int s0_P;
	Int uu_u;
	Int uv_v;
	Int vu_u;
	Int vv_v;
	Int uu_r;
	Int uv_s;
	Int vu_r;
	Int vv_s;

	int64_t bitCount;
	int64_t uu, uv, vu, vv;
	int64_t v0, u0;
	int64_t nb0;

	while (!u.IsZero()) {

		// u' = (uu*u + uv*v) >> bitCount
		// v' = (vu*u + vv*v) >> bitCount
		// Do not maintain a matrix for r and s, the number of 
		// 'added P' can be easily calculated
		uu = 1; uv = 0;
		vu = 0; vv = 1;

		u0 = (int64_t)u.bits64[0];
		v0 = (int64_t)v.bits64[0];
		bitCount = 0;

		// Slightly optimized Binary XCD loop on native signed integers
		// Stop at 62 bits to avoid uv matrix overfow and loss of sign bit
		while (true) {

			while (IS_EVEN(u0) && bitCount < 62) {

				bitCount++;
				u0 >>= 1;
				vu <<= 1;
				vv <<= 1;

			}

			if (bitCount == 62)
				break;

			nb0 = (v0 + u0) & 0x3;
			if (nb0 == 0) {
				SWAP_ADD(uv, vv);
				SWAP_ADD(uu, vu);
				SWAP_ADD(u0, v0);
			}
			else {
				SWAP_SUB(uv, vv);
				SWAP_SUB(uu, vu);
				SWAP_SUB(u0, v0);
			}

		}

		// Now update BigInt variables

		uu_u.IMult(&u, uu);
		uv_v.IMult(&v, uv);

		vu_u.IMult(&u, vu);
		vv_v.IMult(&v, vv);

		uu_r.IMult(&r, uu);
		uv_s.IMult(&s, uv);

		vu_r.IMult(&r, vu);
		vv_s.IMult(&s, vv);

		// Compute multiple of P to add to s and r to make them multiple of 2^62
		uint64_t r0 = ((uu_r.bits64[0] + uv_s.bits64[0]) * MM64) & MSK62;
		uint64_t s0 = ((vu_r.bits64[0] + vv_s.bits64[0]) * MM64) & MSK62;
		r0_P.Mult(&_P, r0);
		s0_P.Mult(&_P, s0);

		// u = (uu*u + uv*v)
		u.Add(&uu_u, &uv_v);

		// v = (vu*u + vv*v)
		v.Add(&vu_u, &vv_v);

		// r = (uu*r + uv*s + r0*P)
		r.Add(&uu_r, &uv_s);
		r.Add(&r0_P);

		// s = (vu*r + vv*s + s0*P)
		s.Add(&vu_r, &vv_s);
		s.Add(&s0_P);

		// Right shift all variables by 62bits
		shiftR(62, u.bits64);
		shiftR(62, v.bits64);
		shiftR(62, r.bits64);
		shiftR(62, s.bits64);

	}

	// v ends with -1 or 1
	if (v.IsNegative()) {
		// V = -1
		v.Neg();
		s.Neg();
		s.Add(&_P);
	}
	if (!v.IsOne()) {
		// No inverse
		CLEAR();
		return;
	}

	if (s.IsNegative())
		s.Add(&_P);

	if (s.IsGreaterOrEqual(&_P))
		s.Sub(&_P);

	Set(&s);

#endif

}

// ------------------------------------------------

void Int::ModExp(Int* e) {

	Int base(this);
	SetInt32(1);
	uint32_t i = 0;

	uint32_t nbBit = e->GetBitLength();
	for (int i = 0; i < (int)nbBit; i++) {
		if (e->GetBit(i))
			ModMul(&base);
		base.ModMul(&base);
	}

}

// ------------------------------------------------

void Int::ModMul(Int* a) {

	Int p;
	p.MontgomeryMult(a, this);
	MontgomeryMult(&_R2, &p);

}

// ------------------------------------------------

void Int::ModSquare(Int* a) {

	Int p;
	p.MontgomeryMult(a, a);
	MontgomeryMult(&_R2, &p);

}

// ------------------------------------------------

void Int::ModCube(Int* a) {

	Int p;
	Int p2;
	p.MontgomeryMult(a, a);
	p2.MontgomeryMult(&p, a);
	MontgomeryMult(&_R3, &p2);

}

// ------------------------------------------------

bool Int::HasSqrt() {

	// Euler's criterion
	Int e(&_P);
	Int a(this);
	e.SubOne();
	e.ShiftR(1);
	a.ModExp(&e);

	return a.IsOne();

}

// ------------------------------------------------

void Int::ModSqrt() {

	if (_P.IsEven()) {
		CLEAR();
		return;
	}

	if (!HasSqrt()) {
		CLEAR();
		return;
	}

	if ((_P.bits64[0] & 3) == 3) {

		Int e(&_P);
		e.AddOne();
		e.ShiftR(2);
		ModExp(&e);

	}
	else if ((_P.bits64[0] & 3) == 1) {

		int nbBit = _P.GetBitLength();

		// Tonelli Shanks
		uint64_t e = 0;
		Int S(&_P);
		S.SubOne();
		while (S.IsEven()) {
			S.ShiftR(1);
			e++;
		}

		// Search smalest non-qresidue of P
		Int q((uint64_t)1);
		do {
			q.AddOne();
		} while (q.HasSqrt());

		Int c(&q);
		c.ModExp(&S);

		Int t(this);
		t.ModExp(&S);

		Int r(this);
		Int ex(&S);
		ex.AddOne();
		ex.ShiftR(1);
		r.ModExp(&ex);

		uint64_t M = e;
		while (!t.IsOne()) {

			Int t2(&t);
			uint64_t i = 0;
			while (!t2.IsOne()) {
				t2.ModSquare(&t2);
				i++;
			}

			Int b(&c);
			for (uint64_t j = 0; j < M - i - 1; j++)
				b.ModSquare(&b);
			M = i;
			c.ModSquare(&b);
			t.ModMul(&t, &c);
			r.ModMul(&r, &b);

		}

		Set(&r);

	}

}

// ------------------------------------------------

void Int::ModMul(Int* a, Int* b) {

	Int p;
	p.MontgomeryMult(a, b);
	MontgomeryMult(&_R2, &p);

}

// ------------------------------------------------

Int* Int::GetFieldCharacteristic() {
	return &_P;
}

// ------------------------------------------------

Int* Int::GetR() {
	return &_R;
}
Int* Int::GetR2() {
	return &_R2;
}
Int* Int::GetR3() {
	return &_R3;
}
Int* Int::GetR4() {
	return &_R4;
}

// ------------------------------------------------

void Int::SetupField(Int* n, Int* R, Int* R2, Int* R3, Int* R4) {

	// Size in number of 32bit word
	int nSize = n->GetSize();

	// Last digit inversions (Newton's iteration)
	{
		int64_t x, t;
		x = t = (int64_t)n->bits64[0];
		x = x * (2 - t * x);
		x = x * (2 - t * x);
		x = x * (2 - t * x);
		x = x * (2 - t * x);
		x = x * (2 - t * x);
		MM64 = (uint64_t)(-x);
		MM32 = (uint32_t)MM64;
	}
	_P.Set(n);

	// Size of Montgomery mult (64bits digit)
	Msize = nSize / 2;

	// Compute few power of R
	// R = 2^(64*Msize) mod n
	Int Ri;
	Ri.MontgomeryMult(&_ONE, &_ONE); // Ri = R^-1
	_R.Set(&Ri);                     // R  = R^-1
	_R2.MontgomeryMult(&Ri, &_ONE);  // R2 = R^-2
	_R3.MontgomeryMult(&Ri, &Ri);    // R3 = R^-3
	_R4.MontgomeryMult(&_R3, &_ONE); // R4 = R^-4

	_R.ModInv();                     // R  = R
	_R2.ModInv();                    // R2 = R^2
	_R3.ModInv();                    // R3 = R^3
	_R4.ModInv();                    // R4 = R^4

	if (R)
		R->Set(&_R);

	if (R2)
		R2->Set(&_R2);

	if (R3)
		R3->Set(&_R3);

	if (R4)
		R4->Set(&_R4);

}
// ------------------------------------------------

uint64_t Int::AddC(Int* a) {

	unsigned char c = 0;
	c = _addcarry_u64(c, bits64[0], a->bits64[0], bits64 + 0);
	c = _addcarry_u64(c, bits64[1], a->bits64[1], bits64 + 1);
	c = _addcarry_u64(c, bits64[2], a->bits64[2], bits64 + 2);
	c = _addcarry_u64(c, bits64[3], a->bits64[3], bits64 + 3);
	c = _addcarry_u64(c, bits64[4], a->bits64[4], bits64 + 4);
#if NB64BLOCK > 5
	c = _addcarry_u64(c, bits64[5], a->bits64[5], bits64 + 5);
	c = _addcarry_u64(c, bits64[6], a->bits64[6], bits64 + 6);
	c = _addcarry_u64(c, bits64[7], a->bits64[7], bits64 + 7);
	c = _addcarry_u64(c, bits64[8], a->bits64[8], bits64 + 8);
#endif

	return c;

}

// ------------------------------------------------

void Int::AddAndShift(Int* a, Int* b, uint64_t cH) {

	unsigned char c = 0;
	c = _addcarry_u64(c, b->bits64[0], a->bits64[0], bits64 + 0);
	c = _addcarry_u64(c, b->bits64[1], a->bits64[1], bits64 + 0);
	c = _addcarry_u64(c, b->bits64[2], a->bits64[2], bits64 + 1);
	c = _addcarry_u64(c, b->bits64[3], a->bits64[3], bits64 + 2);
	c = _addcarry_u64(c, b->bits64[4], a->bits64[4], bits64 + 3);
#if NB64BLOCK > 5
	c = _addcarry_u64(c, b->bits64[5], a->bits64[5], bits64 + 4);
	c = _addcarry_u64(c, b->bits64[6], a->bits64[6], bits64 + 5);
	c = _addcarry_u64(c, b->bits64[7], a->bits64[7], bits64 + 6);
	c = _addcarry_u64(c, b->bits64[8], a->bits64[8], bits64 + 7);
#endif

	bits64[NB64BLOCK - 1] = c + cH;

}

// ------------------------------------------------
void Int::MontgomeryMult(Int* a) {

	// Compute a*b*R^-1 (mod n),  R=2^k (mod n), k = Msize*64
	// a and b must be lower than n
	// See SetupField()

	Int t;
	Int pr;
	Int p;
	uint64_t ML;
	uint64_t c;

	// i = 0
	imm_umul(a->bits64, bits64[0], pr.bits64);
	ML = pr.bits64[0] * MM64;
	imm_umul(_P.bits64, ML, p.bits64);
	c = pr.AddC(&p);
	memcpy(t.bits64, pr.bits64 + 1, 8 * (NB64BLOCK - 1));
	t.bits64[NB64BLOCK - 1] = c;

	for (int i = 1; i < Msize; i++) {

		imm_umul(a->bits64, bits64[i], pr.bits64);
		ML = (pr.bits64[0] + t.bits64[0]) * MM64;
		imm_umul(_P.bits64, ML, p.bits64);
		c = pr.AddC(&p);
		t.AddAndShift(&t, &pr, c);

	}

	p.Sub(&t, &_P);
	if (p.IsPositive())
		Set(&p);
	else
		Set(&t);

}

void Int::MontgomeryMult(Int* a, Int* b) {

	// Compute a*b*R^-1 (mod n),  R=2^k (mod n), k = Msize*64
	// a and b must be lower than n
	// See SetupField()

	Int pr;
	Int p;
	uint64_t ML;
	uint64_t c;

	// i = 0
	imm_umul(a->bits64, b->bits64[0], pr.bits64);
	ML = pr.bits64[0] * MM64;
	imm_umul(_P.bits64, ML, p.bits64);
	c = pr.AddC(&p);
	memcpy(bits64, pr.bits64 + 1, 8 * (NB64BLOCK - 1));
	bits64[NB64BLOCK - 1] = c;

	for (int i = 1; i < Msize; i++) {

		imm_umul(a->bits64, b->bits64[i], pr.bits64);
		ML = (pr.bits64[0] + bits64[0]) * MM64;
		imm_umul(_P.bits64, ML, p.bits64);
		c = pr.AddC(&p);
		AddAndShift(this, &pr, c);

	}

	p.Sub(this, &_P);
	if (p.IsPositive())
		Set(&p);

}


// SecpK1 specific section -----------------------------------------------------------------------------

void Int::ModMulK1(Int* a, Int* b) {

#ifndef WIN64
#if (__GNUC__ > 7) || (__GNUC__ == 7 && (__GNUC_MINOR__ > 2))
	unsigned char c;
#else
	#warning "GCC lass than 7.3 detected, upgrade gcc to get best perfromance"
		volatile unsigned char c;
#endif
#else
	unsigned char c;
#endif


	uint64_t ah, al;
	uint64_t t[5];
	uint64_t r512[8];
	r512[5] = 0;
	r512[6] = 0;
	r512[7] = 0;

	// 256*256 multiplier
	imm_umul(a->bits64, b->bits64[0], r512);
	imm_umul(a->bits64, b->bits64[1], t);
	c = _addcarry_u64(0, r512[1], t[0], r512 + 1);
	c = _addcarry_u64(c, r512[2], t[1], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[2], r512 + 3);
	c = _addcarry_u64(c, r512[4], t[3], r512 + 4);
	c = _addcarry_u64(c, r512[5], t[4], r512 + 5);
	imm_umul(a->bits64, b->bits64[2], t);
	c = _addcarry_u64(0, r512[2], t[0], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[1], r512 + 3);
	c = _addcarry_u64(c, r512[4], t[2], r512 + 4);
	c = _addcarry_u64(c, r512[5], t[3], r512 + 5);
	c = _addcarry_u64(c, r512[6], t[4], r512 + 6);
	imm_umul(a->bits64, b->bits64[3], t);
	c = _addcarry_u64(0, r512[3], t[0], r512 + 3);
	c = _addcarry_u64(c, r512[4], t[1], r512 + 4);
	c = _addcarry_u64(c, r512[5], t[2], r512 + 5);
	c = _addcarry_u64(c, r512[6], t[3], r512 + 6);
	c = _addcarry_u64(c, r512[7], t[4], r512 + 7);

	// Reduce from 512 to 320 
	imm_umul(r512 + 4, 0x1000003D1ULL, t);
	c = _addcarry_u64(0, r512[0], t[0], r512 + 0);
	c = _addcarry_u64(c, r512[1], t[1], r512 + 1);
	c = _addcarry_u64(c, r512[2], t[2], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[3], r512 + 3);

	// Reduce from 320 to 256 
	// No overflow possible here t[4]+c<=0x1000003D1ULL
	al = _umul128(t[4] + c, 0x1000003D1ULL, &ah);
	c = _addcarry_u64(0, r512[0], al, bits64 + 0);
	c = _addcarry_u64(c, r512[1], ah, bits64 + 1);
	c = _addcarry_u64(c, r512[2], 0ULL, bits64 + 2);
	c = _addcarry_u64(c, r512[3], 0ULL, bits64 + 3);

	// Probability of carry here or that this>P is very very unlikely
	bits64[4] = 0;

}

void Int::ModMulK1(Int* a) {

#ifndef WIN64
#if (__GNUC__ > 7) || (__GNUC__ == 7 && (__GNUC_MINOR__ > 2))
	unsigned char c;
#else
	#warning "GCC lass than 7.3 detected, upgrade gcc to get best perfromance"
		volatile unsigned char c;
#endif
#else
	unsigned char c;
#endif

	uint64_t ah, al;
	uint64_t t[5];
	uint64_t r512[8];
	r512[5] = 0;
	r512[6] = 0;
	r512[7] = 0;

	// 256*256 multiplier
	imm_umul(a->bits64, bits64[0], r512);
	imm_umul(a->bits64, bits64[1], t);
	c = _addcarry_u64(0, r512[1], t[0], r512 + 1);
	c = _addcarry_u64(c, r512[2], t[1], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[2], r512 + 3);
	c = _addcarry_u64(c, r512[4], t[3], r512 + 4);
	c = _addcarry_u64(c, r512[5], t[4], r512 + 5);
	imm_umul(a->bits64, bits64[2], t);
	c = _addcarry_u64(0, r512[2], t[0], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[1], r512 + 3);
	c = _addcarry_u64(c, r512[4], t[2], r512 + 4);
	c = _addcarry_u64(c, r512[5], t[3], r512 + 5);
	c = _addcarry_u64(c, r512[6], t[4], r512 + 6);
	imm_umul(a->bits64, bits64[3], t);
	c = _addcarry_u64(0, r512[3], t[0], r512 + 3);
	c = _addcarry_u64(c, r512[4], t[1], r512 + 4);
	c = _addcarry_u64(c, r512[5], t[2], r512 + 5);
	c = _addcarry_u64(c, r512[6], t[3], r512 + 6);
	c = _addcarry_u64(c, r512[7], t[4], r512 + 7);

	// Reduce from 512 to 320 
	imm_umul(r512 + 4, 0x1000003D1ULL, t);
	c = _addcarry_u64(0, r512[0], t[0], r512 + 0);
	c = _addcarry_u64(c, r512[1], t[1], r512 + 1);
	c = _addcarry_u64(c, r512[2], t[2], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[3], r512 + 3);

	// Reduce from 320 to 256 
	// No overflow possible here t[4]+c<=0x1000003D1ULL
	al = _umul128(t[4] + c, 0x1000003D1ULL, &ah);
	c = _addcarry_u64(0, r512[0], al, bits64 + 0);
	c = _addcarry_u64(c, r512[1], ah, bits64 + 1);
	c = _addcarry_u64(c, r512[2], 0, bits64 + 2);
	c = _addcarry_u64(c, r512[3], 0, bits64 + 3);
	// Probability of carry here or that this>P is very very unlikely
	bits64[4] = 0;

}

void Int::ModSquareK1(Int* a) {

#ifndef WIN64
#if (__GNUC__ > 7) || (__GNUC__ == 7 && (__GNUC_MINOR__ > 2))
	unsigned char c;
#else
	#warning "GCC lass than 7.3 detected, upgrade gcc to get best perfromance"
		volatile unsigned char c;
#endif
#else
	unsigned char c;
#endif

	uint64_t r512[8];
	uint64_t u10, u11;
	uint64_t t1;
	uint64_t t2;
	uint64_t t[5];


	//k=0
	r512[0] = _umul128(a->bits64[0], a->bits64[0], &t[1]);

	//k=1
	t[3] = _umul128(a->bits64[0], a->bits64[1], &t[4]);
	c = _addcarry_u64(0, t[3], t[3], &t[3]);
	c = _addcarry_u64(c, t[4], t[4], &t[4]);
	c = _addcarry_u64(c, 0, 0, &t1);
	c = _addcarry_u64(0, t[1], t[3], &t[3]);
	c = _addcarry_u64(c, t[4], 0, &t[4]);
	c = _addcarry_u64(c, t1, 0, &t1);
	r512[1] = t[3];

	//k=2
	t[0] = _umul128(a->bits64[0], a->bits64[2], &t[1]);
	c = _addcarry_u64(0, t[0], t[0], &t[0]);
	c = _addcarry_u64(c, t[1], t[1], &t[1]);
	c = _addcarry_u64(c, 0, 0, &t2);

	u10 = _umul128(a->bits64[1], a->bits64[1], &u11);
	c = _addcarry_u64(0, t[0], u10, &t[0]);
	c = _addcarry_u64(c, t[1], u11, &t[1]);
	c = _addcarry_u64(c, t2, 0, &t2);
	c = _addcarry_u64(0, t[0], t[4], &t[0]);
	c = _addcarry_u64(c, t[1], t1, &t[1]);
	c = _addcarry_u64(c, t2, 0, &t2);
	r512[2] = t[0];

	//k=3
	t[3] = _umul128(a->bits64[0], a->bits64[3], &t[4]);
	u10 = _umul128(a->bits64[1], a->bits64[2], &u11);

	c = _addcarry_u64(0, t[3], u10, &t[3]);
	c = _addcarry_u64(c, t[4], u11, &t[4]);
	c = _addcarry_u64(c, 0, 0, &t1);
	t1 += t1;
	c = _addcarry_u64(0, t[3], t[3], &t[3]);
	c = _addcarry_u64(c, t[4], t[4], &t[4]);
	c = _addcarry_u64(c, t1, 0, &t1);
	c = _addcarry_u64(0, t[3], t[1], &t[3]);
	c = _addcarry_u64(c, t[4], t2, &t[4]);
	c = _addcarry_u64(c, t1, 0, &t1);
	r512[3] = t[3];

	//k=4
	t[0] = _umul128(a->bits64[1], a->bits64[3], &t[1]);
	c = _addcarry_u64(0, t[0], t[0], &t[0]);
	c = _addcarry_u64(c, t[1], t[1], &t[1]);
	c = _addcarry_u64(c, 0, 0, &t2);

	u10 = _umul128(a->bits64[2], a->bits64[2], &u11);
	c = _addcarry_u64(0, t[0], u10, &t[0]);
	c = _addcarry_u64(c, t[1], u11, &t[1]);
	c = _addcarry_u64(c, t2, 0, &t2);
	c = _addcarry_u64(0, t[0], t[4], &t[0]);
	c = _addcarry_u64(c, t[1], t1, &t[1]);
	c = _addcarry_u64(c, t2, 0, &t2);
	r512[4] = t[0];

	//k=5
	t[3] = _umul128(a->bits64[2], a->bits64[3], &t[4]);
	c = _addcarry_u64(0, t[3], t[3], &t[3]);
	c = _addcarry_u64(c, t[4], t[4], &t[4]);
	c = _addcarry_u64(c, 0, 0, &t1);
	c = _addcarry_u64(0, t[3], t[1], &t[3]);
	c = _addcarry_u64(c, t[4], t2, &t[4]);
	c = _addcarry_u64(c, t1, 0, &t1);
	r512[5] = t[3];

	//k=6
	t[0] = _umul128(a->bits64[3], a->bits64[3], &t[1]);
	c = _addcarry_u64(0, t[0], t[4], &t[0]);
	c = _addcarry_u64(c, t[1], t1, &t[1]);
	r512[6] = t[0];

	//k=7
	r512[7] = t[1];

	// Reduce from 512 to 320 
	// Reduce from 512 to 320 
	imm_umul(r512 + 4, 0x1000003D1ULL, t);
	c = _addcarry_u64(0, r512[0], t[0], r512 + 0);
	c = _addcarry_u64(c, r512[1], t[1], r512 + 1);
	c = _addcarry_u64(c, r512[2], t[2], r512 + 2);
	c = _addcarry_u64(c, r512[3], t[3], r512 + 3);

	// Reduce from 320 to 256 
	// No overflow possible here t[4]+c<=0x1000003D1ULL
	u10 = _umul128(t[4] + c, 0x1000003D1ULL, &u11);
	c = _addcarry_u64(0, r512[0], u10, bits64 + 0);
	c = _addcarry_u64(c, r512[1], u11, bits64 + 1);
	c = _addcarry_u64(c, r512[2], 0, bits64 + 2);
	c = _addcarry_u64(c, r512[3], 0, bits64 + 3);
	// Probability of carry here or that this>P is very very unlikely
	bits64[4] = 0;

}

static Int _R2o;                               // R^2 for SecpK1 order modular mult
static uint64_t MM64o = 0x4B0DFF665588B13FULL; // 64bits lsb negative inverse of SecpK1 order
static Int* _O;                                // SecpK1 order

void Int::InitK1(Int* order) {
	_O = order;
	_R2o.SetBase16("9D671CD581C69BC5E697F5E45BCD07C6741496C20E7CF878896CF21467D7D140");
}

void Int::ModAddK1order(Int* a, Int* b) {
	Add(a, b);
	Sub(_O);
	if (IsNegative())
		Add(_O);
}

void Int::ModAddK1order(Int* a) {
	Add(a);
	Sub(_O);
	if (IsNegative())
		Add(_O);
}

void Int::ModSubK1order(Int* a) {
	Sub(a);
	if (IsNegative())
		Add(_O);
}

void Int::ModNegK1order() {
	Neg();
	Add(_O);
}

uint32_t Int::ModPositiveK1() {

	Int N(this);
	Int D(this);
	N.ModNeg();
	D.Sub(&N);
	if (D.IsNegative()) {
		return 0;
	}
	else {
		Set(&N);
		return 1;
	}

}


void Int::ModMulK1order(Int* a) {

	Int t;
	Int pr;
	Int p;
	uint64_t ML;
	uint64_t c;

	imm_umul(a->bits64, bits64[0], pr.bits64);
	ML = pr.bits64[0] * MM64o;
	imm_umul(_O->bits64, ML, p.bits64);
	c = pr.AddC(&p);
	memcpy(t.bits64, pr.bits64 + 1, 32);
	t.bits64[4] = c;

	for (int i = 1; i < 4; i++) {

		imm_umul(a->bits64, bits64[i], pr.bits64);
		ML = (pr.bits64[0] + t.bits64[0]) * MM64o;
		imm_umul(_O->bits64, ML, p.bits64);
		c = pr.AddC(&p);
		t.AddAndShift(&t, &pr, c);

	}

	p.Sub(&t, _O);
	if (p.IsPositive())
		Set(&p);
	else
		Set(&t);


	// Normalize

	imm_umul(_R2o.bits64, bits64[0], pr.bits64);
	ML = pr.bits64[0] * MM64o;
	imm_umul(_O->bits64, ML, p.bits64);
	c = pr.AddC(&p);
	memcpy(t.bits64, pr.bits64 + 1, 32);
	t.bits64[4] = c;

	for (int i = 1; i < 4; i++) {

		imm_umul(_R2o.bits64, bits64[i], pr.bits64);
		ML = (pr.bits64[0] + t.bits64[0]) * MM64o;
		imm_umul(_O->bits64, ML, p.bits64);
		c = pr.AddC(&p);
		t.AddAndShift(&t, &pr, c);

	}

	p.Sub(&t, _O);
	if (p.IsPositive())
		Set(&p);
	else
		Set(&t);

}
#ifndef LOSTCOINSH
#define LOSTCOINSH

#include <string>
#include <vector>
#include "SECP256k1.h"
#include "Bloom.h"
#include "GPU/GPUEngine.h"
#ifdef WIN64
#include <Windows.h>
#endif

#define CPU_GRP_SIZE 1024

class LostCoins;

typedef struct {

	LostCoins* obj;
	int  threadId;
	bool isRunning;
	bool hasStarted;
	bool rekeyRequest;
	int  gridSizeX;
	int  gridSizeY;
	int  gpuId;

} TH_PARAM;


class LostCoins
{

public:

	LostCoins(std::string addressFile, std::string seed, std::string zez, int diz, int searchMode,
		bool useGpu, std::string outputFile, bool useSSE, uint32_t maxFound,
		uint64_t rekey, int nbit, bool paranoiacSeed, const std::string& rangeStart1, const std::string& rangeEnd1, bool& should_exit);
	~LostCoins();

	void Search(int nbThread, std::vector<int> gpuId, std::vector<int> gridSize, bool& should_exit);
	void FindKeyCPU(TH_PARAM* p);
	void FindKeyGPU(TH_PARAM* p);

private:

	std::string GetHex(std::vector<unsigned char>& buffer);
	bool checkPrivKey(std::string addr, Int& key, int32_t incr, int endomorphism, bool mode);
	void checkAddresses(bool compressed, Int key, int i, Point p1);
	void checkAddressesSSE(bool compressed, Int key, int i, Point p1, Point p2, Point p3, Point p4);
	void output(std::string addr, std::string pAddr, std::string pAddrHex);
	bool isAlive(TH_PARAM* p);

	bool hasStarted(TH_PARAM* p);
	void rekeyRequest(TH_PARAM* p);
	uint64_t getGPUCount();
	uint64_t getCPUCount();
	void getCPUStartingKey(int thId, Int& key, Point& startP);
	void getGPUStartingKeys(int thId, int groupSize, int nbThread, Int* keys, Point* p);

	int CheckBloomBinary(const uint8_t* hash);
	std::string formatThousands(uint64_t x);
	char* toTimeStr(int sec, char* timeStr);

	Secp256K1* secp;
	Bloom* bloom;

	Int startKey;
	uint64_t counters[256];
	double startTime;

	int searchMode;
	int searchType;

	bool useGpu;
	bool endOfSearch;
	int nbCPUThread;
	int nbGPUThread;
	int nbFoundKey;
	int nbit;
	int diz;
	uint64_t rekey;
	uint64_t lastRekey;
	std::string outputFile;
	std::string seed;
	std::string zez;
	std::string addressFile;
	bool useSSE;
	Int rangeStart1;
	Int rangeEnd1;
	Int rangeDiff;
	Int rangeDiff2;
	Int rangeDiff3;
	Int key22;
	uint32_t maxFound;

	uint8_t* DATA;
	uint64_t TOTAL_ADDR;
	uint64_t BLOOM_N;

	Int beta;
	Int lambda;
	Int beta2;
	Int lambda2;

#ifdef WIN64
	HANDLE ghMutex;
#else
	pthread_mutex_t  ghMutex;
#endif

};

#endif // LOSTCOINSH
<?xml version="1.0" encoding="utf-8"?>
<Project DefaultTargets="Build" ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <ItemGroup Label="ProjectConfigurations">
    <ProjectConfiguration Include="Debug|x64">
      <Configuration>Debug</Configuration>
      <Platform>x64</Platform>
    </ProjectConfiguration>
    <ProjectConfiguration Include="Release|x64">
      <Configuration>Release</Configuration>
      <Platform>x64</Platform>
    </ProjectConfiguration>
  </ItemGroup>
  <ItemGroup>
    <ClCompile Include="Base58.cpp" />
    <ClCompile Include="Bech32.cpp" />
    <ClCompile Include="Bloom.cpp" />
    <ClCompile Include="GPU\GPUGenerate.cpp" />
    <ClCompile Include="hash\ripemd160.cpp" />
    <ClCompile Include="hash\ripemd160_sse.cpp" />
    <ClCompile Include="hash\sha256.cpp" />
    <ClCompile Include="hash\sha256_sse.cpp" />
    <ClCompile Include="hash\sha512.cpp" />
    <ClCompile Include="Int.cpp" />
    <ClCompile Include="IntGroup.cpp" />
    <ClCompile Include="IntMod.cpp" />
    <ClCompile Include="LostCoins.cpp" />
    <ClCompile Include="Main.cpp" />
    <ClCompile Include="Point.cpp" />
    <ClCompile Include="Random.cpp" />
    <ClCompile Include="SECP256K1.cpp" />
    <ClCompile Include="Timer.cpp" />
  </ItemGroup>
  <ItemGroup>
    <ClInclude Include="ArgParse.h" />
    <ClInclude Include="Base58.h" />
    <ClInclude Include="Bech32.h" />
    <ClInclude Include="Bloom.h" />
    <ClInclude Include="GPU\GPUBase58.h" />
    <ClInclude Include="GPU\GPUCompute.h" />
    <ClInclude Include="GPU\GPUEngine.h" />
    <ClInclude Include="GPU\GPUGroup.h" />
    <ClInclude Include="GPU\GPUHash.h" />
    <ClInclude Include="GPU\GPUMath.h" />
    <ClInclude Include="hash\ripemd160.h" />
    <ClInclude Include="hash\sha256.h" />
    <ClInclude Include="hash\sha512.h" />
    <ClInclude Include="Int.h" />
    <ClInclude Include="IntGroup.h" />
    <ClInclude Include="LostCoins.h" />
    <ClInclude Include="Point.h" />
    <ClInclude Include="Random.h" />
    <ClInclude Include="SECP256k1.h" />
    <ClInclude Include="Timer.h" />
  </ItemGroup>
  <ItemGroup>
    <CudaCompile Include="GPU\GPUEngine.cu" />
  </ItemGroup>
  <PropertyGroup Label="Globals">
    <ProjectGuid>{C982E936-E84D-40E6-AA84-D439DC2D83A9}</ProjectGuid>
    <RootNamespace>LostCoins</RootNamespace>
    <WindowsTargetPlatformVersion>10.0</WindowsTargetPlatformVersion>
  </PropertyGroup>
  <Import Project="$(VCTargetsPath)\Microsoft.Cpp.Default.props" />
  <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|x64'" Label="Configuration">
    <ConfigurationType>Application</ConfigurationType>
    <UseDebugLibraries>true</UseDebugLibraries>
    <CharacterSet>MultiByte</CharacterSet>
    <PlatformToolset>v142</PlatformToolset>
  </PropertyGroup>
  <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Release|x64'" Label="Configuration">
    <ConfigurationType>Application</ConfigurationType>
    <UseDebugLibraries>false</UseDebugLibraries>
    <WholeProgramOptimization>true</WholeProgramOptimization>
    <CharacterSet>MultiByte</CharacterSet>
    <PlatformToolset>v142</PlatformToolset>
  </PropertyGroup>
  <Import Project="$(VCTargetsPath)\Microsoft.Cpp.props" />
  <ImportGroup Label="ExtensionSettings">
    <Import Project="$(VCTargetsPath)\BuildCustomizations\CUDA 10.2.props" />
  </ImportGroup>
  <ImportGroup Label="PropertySheets" Condition="'$(Configuration)|$(Platform)'=='Debug|x64'">
    <Import Project="$(UserRootDir)\Microsoft.Cpp.$(Platform).user.props" Condition="exists('$(UserRootDir)\Microsoft.Cpp.$(Platform).user.props')" Label="LocalAppDataPlatform" />
  </ImportGroup>
  <ImportGroup Label="PropertySheets" Condition="'$(Configuration)|$(Platform)'=='Release|x64'">
    <Import Project="$(UserRootDir)\Microsoft.Cpp.$(Platform).user.props" Condition="exists('$(UserRootDir)\Microsoft.Cpp.$(Platform).user.props')" Label="LocalAppDataPlatform" />
  </ImportGroup>
  <PropertyGroup Label="UserMacros" />
  <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|x64'">
    <LinkIncremental>true</LinkIncremental>
  </PropertyGroup>
  <ItemDefinitionGroup Condition="'$(Configuration)|$(Platform)'=='Debug|x64'">
    <ClCompile>
      <WarningLevel>Level3</WarningLevel>
      <Optimization>Disabled</Optimization>
      <PreprocessorDefinitions>WIN32;WIN64;_DEBUG;_CONSOLE;WITHGPU;_CRT_SECURE_NO_WARNINGS;%(PreprocessorDefinitions)</PreprocessorDefinitions>
      <EnableEnhancedInstructionSet>StreamingSIMDExtensions</EnableEnhancedInstructionSet>
    </ClCompile>
    <Link>
      <GenerateDebugInformation>true</GenerateDebugInformation>
      <SubSystem>Console</SubSystem>
      <AdditionalDependencies>cudart_static.lib;kernel32.lib;user32.lib;gdi32.lib;winspool.lib;comdlg32.lib;advapi32.lib;shell32.lib;ole32.lib;oleaut32.lib;uuid.lib;odbc32.lib;odbccp32.lib;%(AdditionalDependencies)</AdditionalDependencies>
    </Link>
    <CudaCompile>
      <TargetMachinePlatform>64</TargetMachinePlatform>
    </CudaCompile>
  </ItemDefinitionGroup>
  <ItemDefinitionGroup Condition="'$(Configuration)|$(Platform)'=='Release|x64'">
    <ClCompile>
      <WarningLevel>Level3</WarningLevel>
      <Optimization>MaxSpeed</Optimization>
      <FunctionLevelLinking>true</FunctionLevelLinking>
      <IntrinsicFunctions>true</IntrinsicFunctions>
      <PreprocessorDefinitions>WIN32;WIN64;NDEBUG;_CONSOLE;WITHGPU;_CRT_SECURE_NO_WARNINGS;%(PreprocessorDefinitions)</PreprocessorDefinitions>
      <EnableEnhancedInstructionSet>StreamingSIMDExtensions</EnableEnhancedInstructionSet>
      <InlineFunctionExpansion>AnySuitable</InlineFunctionExpansion>
      <FavorSizeOrSpeed>Speed</FavorSizeOrSpeed>
    </ClCompile>
    <Link>
      <GenerateDebugInformation>true</GenerateDebugInformation>
      <EnableCOMDATFolding>true</EnableCOMDATFolding>
      <OptimizeReferences>true</OptimizeReferences>
      <SubSystem>Console</SubSystem>
      <AdditionalDependencies>cudart_static.lib;kernel32.lib;user32.lib;gdi32.lib;winspool.lib;comdlg32.lib;advapi32.lib;shell32.lib;ole32.lib;oleaut32.lib;uuid.lib;odbc32.lib;odbccp32.lib;%(AdditionalDependencies)</AdditionalDependencies>
    </Link>
    <CudaCompile>
      <TargetMachinePlatform>64</TargetMachinePlatform>
    </CudaCompile>
  </ItemDefinitionGroup>
  <Import Project="$(VCTargetsPath)\Microsoft.Cpp.targets" />
  <ImportGroup Label="ExtensionTargets">
    <Import Project="$(VCTargetsPath)\BuildCustomizations\CUDA 10.2.targets" />
  </ImportGroup>
</Project>
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <ItemGroup>
    <Filter Include="BLOOM">
      <UniqueIdentifier>{698582e5-391f-4f09-bd52-41c0cefad772}</UniqueIdentifier>
    </Filter>
    <Filter Include="ENCODE">
      <UniqueIdentifier>{9f3ef0d4-08c9-4ebd-80d0-828a43d4773e}</UniqueIdentifier>
    </Filter>
    <Filter Include="GPU">
      <UniqueIdentifier>{8f82e15c-429f-4683-8552-ff7c67fef1b8}</UniqueIdentifier>
    </Filter>
    <Filter Include="HASH">
      <UniqueIdentifier>{cd2d9c02-0c0d-4698-a07f-df2427ede5ab}</UniqueIdentifier>
    </Filter>
    <Filter Include="LostCoins">
      <UniqueIdentifier>{957fb233-6559-48cb-aeb4-842f05530f86}</UniqueIdentifier>
    </Filter>
    <Filter Include="SECP256K1">
      <UniqueIdentifier>{1f2d7638-1fb3-4430-9598-4dafd7cf1364}</UniqueIdentifier>
    </Filter>
  </ItemGroup>
  <ItemGroup>
    <ClCompile Include="Bloom.cpp">
      <Filter>BLOOM</Filter>
    </ClCompile>
    <ClCompile Include="Base58.cpp">
      <Filter>ENCODE</Filter>
    </ClCompile>
    <ClCompile Include="GPU\GPUGenerate.cpp">
      <Filter>GPU</Filter>
    </ClCompile>
    <ClCompile Include="hash\ripemd160.cpp">
      <Filter>HASH</Filter>
    </ClCompile>
    <ClCompile Include="hash\ripemd160_sse.cpp">
      <Filter>HASH</Filter>
    </ClCompile>
    <ClCompile Include="hash\sha256.cpp">
      <Filter>HASH</Filter>
    </ClCompile>
    <ClCompile Include="hash\sha256_sse.cpp">
      <Filter>HASH</Filter>
    </ClCompile>
    <ClCompile Include="hash\sha512.cpp">
      <Filter>HASH</Filter>
    </ClCompile>
    <ClCompile Include="LostCoins.cpp">
      <Filter>KEYHUNT</Filter>
    </ClCompile>
    <ClCompile Include="Timer.cpp">
      <Filter>KEYHUNT</Filter>
    </ClCompile>
    <ClCompile Include="Int.cpp">
      <Filter>SECP256K1</Filter>
    </ClCompile>
    <ClCompile Include="IntGroup.cpp">
      <Filter>SECP256K1</Filter>
    </ClCompile>
    <ClCompile Include="IntMod.cpp">
      <Filter>SECP256K1</Filter>
    </ClCompile>
    <ClCompile Include="Point.cpp">
      <Filter>SECP256K1</Filter>
    </ClCompile>
    <ClCompile Include="Random.cpp">
      <Filter>SECP256K1</Filter>
    </ClCompile>
    <ClCompile Include="SECP256K1.cpp">
      <Filter>SECP256K1</Filter>
    </ClCompile>
    <ClCompile Include="Main.cpp" />
    <ClCompile Include="Bech32.cpp">
      <Filter>ENCODE</Filter>
    </ClCompile>
  </ItemGroup>
  <ItemGroup>
    <ClInclude Include="Bloom.h">
      <Filter>BLOOM</Filter>
    </ClInclude>
    <ClInclude Include="Base58.h">
      <Filter>ENCODE</Filter>
    </ClInclude>
    <ClInclude Include="GPU\GPUBase58.h">
      <Filter>GPU</Filter>
    </ClInclude>
    <ClInclude Include="GPU\GPUCompute.h">
      <Filter>GPU</Filter>
    </ClInclude>
    <ClInclude Include="GPU\GPUEngine.h">
      <Filter>GPU</Filter>
    </ClInclude>
    <ClInclude Include="GPU\GPUGroup.h">
      <Filter>GPU</Filter>
    </ClInclude>
    <ClInclude Include="GPU\GPUHash.h">
      <Filter>GPU</Filter>
    </ClInclude>
    <ClInclude Include="GPU\GPUMath.h">
      <Filter>GPU</Filter>
    </ClInclude>
    <ClInclude Include="hash\ripemd160.h">
      <Filter>HASH</Filter>
    </ClInclude>
    <ClInclude Include="hash\sha256.h">
      <Filter>HASH</Filter>
    </ClInclude>
    <ClInclude Include="hash\sha512.h">
      <Filter>HASH</Filter>
    </ClInclude>
    <ClInclude Include="ArgParse.h">
      <Filter>KEYHUNT</Filter>
    </ClInclude>
    <ClInclude Include="LostCoins.h">
      <Filter>KEYHUNT</Filter>
    </ClInclude>
    <ClInclude Include="Timer.h">
      <Filter>KEYHUNT</Filter>
    </ClInclude>
    <ClInclude Include="Int.h">
      <Filter>SECP256K1</Filter>
    </ClInclude>
    <ClInclude Include="IntGroup.h">
      <Filter>SECP256K1</Filter>
    </ClInclude>
    <ClInclude Include="Point.h">
      <Filter>SECP256K1</Filter>
    </ClInclude>
    <ClInclude Include="Random.h">
      <Filter>SECP256K1</Filter>
    </ClInclude>
    <ClInclude Include="SECP256k1.h">
      <Filter>SECP256K1</Filter>
    </ClInclude>
    <ClInclude Include="Bech32.h">
      <Filter>ENCODE</Filter>
    </ClInclude>
  </ItemGroup>
  <ItemGroup>
    <CudaCompile Include="GPU\GPUEngine.cu">
      <Filter>GPU</Filter>
    </CudaCompile>
  </ItemGroup>
</Project>
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|x64'">
    <LocalDebuggerCommandArguments>-t 0 -g -i 0 -x 256,256 -n 256 -f address1-160-sorted.bin</LocalDebuggerCommandArguments>
    <DebuggerFlavor>WindowsLocalDebugger</DebuggerFlavor>
  </PropertyGroup>
  <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Release|x64'">
    <LocalDebuggerCommandArguments>-c</LocalDebuggerCommandArguments>
    <DebuggerFlavor>WindowsLocalDebugger</DebuggerFlavor>
  </PropertyGroup>
</Project>
#include "Timer.h"
#include "LostCoins.h"
#include "ArgParse.h"
#include <fstream>
#include <string>
#include <string.h>
#include <stdexcept>
#include <iostream>
#include <windows.h>
#define RELEASE "1.0"

using namespace std;
using namespace argparse;
bool should_exit = false;

// ----------------------------------------------------------------------------
// ----------------------------------------------------------------------------

const char* vstr = "Print version. For help visit https://github.com/phrutis/LostCoins                              ";
const char* cstr = "Check the working of the code LostCoins                                                         ";
const char* ustr = "Search only uncompressed addresses                                                              ";
const char* bstr = "Search both (uncompressed and compressed addresses)                                             ";
const char* gstr = "Enable GPU calculation                                                                          ";
const char* istr = "GPU ids: 0,1...: List of GPU(s) to use, default is 0                                            ";
const char* xstr = "GPU gridsize: g0x,g0y,g1x,g1y, ...: Specify GPU(s) kernel gridsize, default is 8*(MP number),128";
const char* ostr = "Outputfile: Output results to the specified file, default: Found.txt                            ";
const char* mstr = "Specify maximun number of addresses found by each kernel call                                   ";
const char* tstr = "threadNumber: Specify number of CPU thread, default is number of core                           ";
const char* estr = "Disable SSE hash function                                                                       ";
const char* lstr = "List cuda enabled devices                                                                       ";
const char* rstr = "Number of random modes                                                                          ";
const char* nstr = "Number of letters or number bit range 1-256                                                     ";
const char* fstr = "RIPEMD160 binary hash file path                                                                 ";
const char* sstr = "PassPhrase   (Start bit range)                                                                  ";
const char* szez = "PassPhrase 2 (End bit range)                                                                    ";
const char* sdiz = "Display modes -d 1 Show hashes, slow speed -d 2 only counters, the fastest speed. -d 0 default  ";
const char* scolor = "- color 1-255 Recommended colors: 3, 10, 11, 14, 15, 240(White-black)                         ";
// ----------------------------------------------------------------------------
// ----------------------------------------------------------------------------

void getInts(string name, vector<int>& tokens, const string& text, char sep)
{

	size_t start = 0, end = 0;
	tokens.clear();
	int item;

	try {

		while ((end = text.find(sep, start)) != string::npos) {
			item = std::stoi(text.substr(start, end - start));
			tokens.push_back(item);
			start = end + 1;
		}

		item = std::stoi(text.substr(start));
		tokens.push_back(item);

	}
	catch (std::invalid_argument&) {

		printf("Invalid %s argument, number expected\n", name.c_str());
		exit(-1);

	}

}

BOOL WINAPI CtrlHandler(DWORD fdwCtrlType)
{
	switch (fdwCtrlType) {
	case CTRL_C_EVENT:
		//printf("\n\nCtrl-C event\n\n");
		should_exit = true;
		return TRUE;

	default:
		return TRUE;
	}
}

int main(int argc, const char* argv[])
{
	// Global Init
	Timer::Init();
	rseed(Timer::getSeed32());
	struct console
	{
		console(unsigned width, unsigned height)
		{
			SMALL_RECT r;
			COORD      c;
			hConOut = GetStdHandle(STD_OUTPUT_HANDLE);
			if (!GetConsoleScreenBufferInfo(hConOut, &csbi))
				throw runtime_error("You must be attached to a human.");

			r.Left =
				r.Top = 0;
			r.Right = width - 1;
			r.Bottom = height - 1;
			SetConsoleWindowInfo(hConOut, TRUE, &r);

			c.X = width;
			c.Y = height;
			SetConsoleScreenBufferSize(hConOut, c);
		}

		~console()
		{
			SetConsoleTextAttribute(hConOut, csbi.wAttributes);
			SetConsoleScreenBufferSize(hConOut, csbi.dwSize);
			SetConsoleWindowInfo(hConOut, TRUE, &csbi.srWindow);
		}

		void color(WORD color = 0x07)
		{
			SetConsoleTextAttribute(hConOut, color);
		}

		HANDLE                     hConOut;
		CONSOLE_SCREEN_BUFFER_INFO csbi;
	};

	//----------------------------------------------------------------------------
	console con(200, 400);

	//----------------------------------------------------------------------------


	bool gpuEnable = false;
	int searchMode = SEARCH_COMPRESSED;
	vector<int> gpuId = { 0 };
	vector<int> gridSize;
	string seed = "";
	string zez = "";
	int diz = 0;
	string outputFile = "Found.txt";
	string hash160File = "";
	int nbCPUThread = Timer::getCoreNumber();
	int nbit = 0;
	int color = 15;
	bool tSpecified = false;
	bool sse = true;
	uint32_t maxFound = 100;
	uint64_t rekey = 0;
	bool paranoiacSeed = false;
	string rangeStart1 = "1";
	string rangeEnd1 = "0";

	ArgumentParser parser("LostCoins", "Hunt for Bitcoin private keys random");

	parser.add_argument("-v", "--version", vstr, false);
	parser.add_argument("-c", "--check", cstr, false);
	parser.add_argument("-u", "--uncomp", ustr, false);
	parser.add_argument("-b", "--both", bstr, false);
	parser.add_argument("-g", "--gpu", gstr, false);
	parser.add_argument("-i", "--gpui", istr, false);
	parser.add_argument("-x", "--gpux", xstr, false);
	parser.add_argument("-o", "--out", ostr, false);
	parser.add_argument("-m", "--max", mstr, false);
	parser.add_argument("-s", "--seed", sstr, false);
	parser.add_argument("-t", "--thread", tstr, false);
	parser.add_argument("-e", "--nosse", estr, false);
	parser.add_argument("-l", "--list", lstr, false);
	parser.add_argument("-r", "--rkey", rstr, false);
	parser.add_argument("-n", "--nbit", nstr, false);
	parser.add_argument("-f", "--file", fstr, false);
	parser.add_argument("-z", "--zez", szez, false);
	parser.add_argument("-d", "--diz", sdiz, false);
	parser.add_argument("-k", "--color", scolor, false);
	parser.enable_help();

	auto err = parser.parse(argc, argv);
	if (err) {
		std::cout << err << std::endl;
		parser.print_help();
		return -1;
	}

	if (parser.exists("help")) {
		parser.print_help();
		return 0;
	}

	if (parser.exists("version")) {
		printf(" LostCoins v" RELEASE "\n");
		return 0;
	}

	if (parser.exists("check")) {
		printf("LostCoins v" RELEASE "\n");
		printf("Checking... Int\n\n");
		Int K;
		K.SetBase16("3EF7CEF65557B61DC4FF2313D0049C584017659A32B002C105D04A19DA52CB47");
		K.Check();

		printf("\n\nChecking... Secp256K1\n\n");
		Secp256K1 sec;
		sec.Init();
		sec.Check();

		return 0;
	}

	if (parser.exists("uncomp")) {
		searchMode = SEARCH_UNCOMPRESSED;
	}
	if (parser.exists("both")) {
		searchMode = SEARCH_BOTH;
	}

	if (parser.exists("gpu")) {
		gpuEnable = true;
	}

	if (parser.exists("gpui")) {
		string ids = parser.get<string>("i");
		getInts("gpui", gpuId, ids, ',');
	}

	if (parser.exists("gpux")) {
		string grids = parser.get<string>("x");
		getInts("gpux", gridSize, grids, ',');
	}

	if (parser.exists("out")) {
		outputFile = parser.get<string>("o");
	}

	if (parser.exists("max")) {
		maxFound = parser.get<uint32_t>("m");
	}

	if (parser.exists("seed")) {
		seed = parser.get<string>("s");
		paranoiacSeed = true;
	}
	if (parser.exists("zez")) {
		zez = parser.get<string>("z");

	}
	if (parser.exists("diz")) {
		diz = parser.get<int>("d");

	}
	if (parser.exists("color")) {
		color = parser.get<int>("k");
	}

	if (parser.exists("thread")) {
		nbCPUThread = parser.get<int>("t");
		tSpecified = true;
	}

	if (parser.exists("nosse")) {
		sse = false;
	}

	if (parser.exists("list")) {
		GPUEngine::PrintCudaInfo();
		return 0;
	}

	if (parser.exists("rkey")) {
		rekey = parser.get<uint64_t>("r");
	}

	if (parser.exists("nbit")) {
		nbit = parser.get<int>("n");
		if (nbit < 0 || nbit > 256) {
			printf("Invalid nbit value, must have in range: 1 - 256\n");
			exit(-1);
		}
	}

	if (parser.exists("file")) {
		hash160File = parser.get<string>("f");
	}


	if (gridSize.size() == 0) {
		for (int i = 0; i < gpuId.size(); i++) {
			gridSize.push_back(-1);
			gridSize.push_back(128);
		}
	}
	else if (gridSize.size() != gpuId.size() * 2) {
		printf("Invalid gridSize or gpuId argument, must have coherent size\n");
		exit(-1);
	}

	if (hash160File.length() <= 0) {
		printf("Invalid RIPEMD160 binary hash file path\n");
		exit(-1);
	}
	HANDLE hConsole = GetStdHandle(STD_OUTPUT_HANDLE);
	SetConsoleTextAttribute(hConsole, color);

	// Let one CPU core free per gpu is gpu is enabled
	// It will avoid to hang the system
	if (!tSpecified && nbCPUThread > 1 && gpuEnable)
		nbCPUThread -= (int)gpuId.size();
	if (nbCPUThread < 0)
		nbCPUThread = 0;

	{
		printf("\n");
		printf(" LostCoins v1.0\n");
		printf("\n");
		printf(" SEARCH MODE  : %s\n", searchMode == SEARCH_COMPRESSED ? "COMPRESSED" : (searchMode == SEARCH_UNCOMPRESSED ? "UNCOMPRESSED" : "COMPRESSED & UNCOMPRESSED"));
		printf(" DEVICE       : %s\n", (gpuEnable && nbCPUThread > 0) ? "CPU & GPU" : ((!gpuEnable && nbCPUThread > 0) ? "CPU" : "GPU"));
		printf(" CPU THREAD   : %d\n", nbCPUThread);
		printf(" GPU IDS      : ");
		for (int i = 0; i < gpuId.size(); i++) {
			printf("%d", gpuId.at(i));
			if (i + 1 < gpuId.size())
				printf(", ");
		}
		printf("\n");
		printf(" GPU GRIDSIZE : ");
		for (int i = 0; i < gridSize.size(); i++) {
			printf("%d", gridSize.at(i));
			if (i + 1 < gridSize.size()) {
				if ((i + 1) % 2 != 0) {
					printf("x");
				}
				else {
					printf(", ");
				}

			}
		}
		char* speed;
		if (diz == 0) {
			speed = "NORMAL";
		}
		if (diz == 1) {
			speed = "SLOW (hashes sha256 are displayed)";
		}
		if (diz > 1) {
			speed = "HIGH";
		}
		setlocale(LC_ALL, "Russian");
		printf("\n");
		printf(" RANDOM MODE  : %llu\n", rekey);
		printf(" ROTOR SPEED  : %s\n", speed);
		printf(" CHARACTERS   : %d\n", nbit);
		printf(" PASSPHRASE   : %s\n", seed.c_str());
		printf(" PASSPHRASE 2 : %s\n", zez.c_str());
		printf(" DISPLAY MODE : %d\n", diz);
		printf(" TEXT COLOR   : %d\n", color);
		printf(" MAX FOUND    : %d\n", maxFound);
		printf(" HASH160 FILE : %s\n", hash160File.c_str());
		printf(" OUTPUT FILE  : %s\n", outputFile.c_str());
	}

	if (SetConsoleCtrlHandler(CtrlHandler, TRUE)) {
		LostCoins* v = new LostCoins(hash160File, seed, zez, diz, searchMode, gpuEnable,
			outputFile, sse, maxFound, rekey, nbit, paranoiacSeed, rangeStart1, rangeEnd1, should_exit);

		v->Search(nbCPUThread, gpuId, gridSize, should_exit);

		delete v;
		printf("\n\nBYE\n");
		return 0;
	}
	else {
		printf("error: could not set control-c handler\n");
		return 1;
	}
}
/*
 * This file is part of the VanitySearch distribution (https://github.com/JeanLucPons/VanitySearch).
 * Copyright (c) 2019 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#include "Point.h"

Point::Point()
{
}

Point::Point(const Point &p)
{
    x.Set((Int *)&p.x);
    y.Set((Int *)&p.y);
    z.Set((Int *)&p.z);
}

Point::Point(Int *cx, Int *cy, Int *cz)
{
    x.Set(cx);
    y.Set(cy);
    z.Set(cz);
}

Point::Point(Int *cx, Int *cz)
{
    x.Set(cx);
    z.Set(cz);
}

void Point::Clear()
{
    x.SetInt32(0);
    y.SetInt32(0);
    z.SetInt32(0);
}

void Point::Set(Int *cx, Int *cy, Int *cz)
{
    x.Set(cx);
    y.Set(cy);
    z.Set(cz);
}

Point::~Point()
{
}

void Point::Set(Point &p)
{
    x.Set(&p.x);
    y.Set(&p.y);
}

bool Point::isZero()
{
    return x.IsZero() && y.IsZero();
}

void Point::Reduce()
{

    Int i(&z);
    i.ModInv();
    x.ModMul(&x, &i);
    y.ModMul(&y, &i);
    z.SetInt32(1);

}

bool Point::equals(Point &p)
{
    return x.IsEqual(&p.x) && y.IsEqual(&p.y) && z.IsEqual(&p.z);
}

std::string Point::toString()
{

    std::string ret;
    ret  = "X=" + x.GetBase16() + "\n";
    ret += "Y=" + y.GetBase16() + "\n";
    ret += "Z=" + z.GetBase16() + "\n";
    return ret;

}
/*
 * This file is part of the VanitySearch distribution (https://github.com/JeanLucPons/VanitySearch).
 * Copyright (c) 2019 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#ifndef POINTH
#define POINTH

#include "Int.h"

class Point
{

public:

    Point();
    Point(Int *cx, Int *cy, Int *cz);
    Point(Int *cx, Int *cz);
    Point(const Point &p);
    ~Point();
    bool isZero();
    bool equals(Point &p);
    void Set(Point &p);
    void Set(Int *cx, Int *cy, Int *cz);
    void Clear();
    void Reduce();
    std::string toString();

    Int x;
    Int y;
    Int z;

};

#endif // POINTH
/*
 * This file is part of the VanitySearch distribution (https://github.com/JeanLucPons/VanitySearch).
 * Copyright (c) 2019 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#include "Random.h"

#define  RK_STATE_LEN 624

/* State of the RNG */
typedef struct rk_state_ {
    unsigned long key[RK_STATE_LEN];
    int pos;
} rk_state;

rk_state localState;

/* Maximum generated random value */
#define RK_MAX 0xFFFFFFFFUL

void rk_seed(unsigned long seed, rk_state *state)
{
    int pos;
    seed &= 0xffffffffUL;

    /* Knuth's PRNG as used in the Mersenne Twister reference implementation */
    for (pos = 0; pos < RK_STATE_LEN; pos++) {
        state->key[pos] = seed;
        seed = (1812433253UL * (seed ^ (seed >> 30)) + pos + 1) & 0xffffffffUL;
    }

    state->pos = RK_STATE_LEN;
}

/* Magic Mersenne Twister constants */
#define N 624
#define M 397
#define MATRIX_A 0x9908b0dfUL
#define UPPER_MASK 0x80000000UL
#define LOWER_MASK 0x7fffffffUL

#ifdef WIN32
// Disable "unary minus operator applied to unsigned type, result still unsigned" warning.
#pragma warning(disable : 4146)
#endif

/* Slightly optimised reference implementation of the Mersenne Twister */
inline unsigned long rk_random(rk_state *state)
{
    unsigned long y;

    if (state->pos == RK_STATE_LEN) {
        int i;

        for (i = 0; i < N - M; i++) {
            y = (state->key[i] & UPPER_MASK) | (state->key[i + 1] & LOWER_MASK);
            state->key[i] = state->key[i + M] ^ (y >> 1) ^ (-(y & 1) & MATRIX_A);
        }
        for (; i < N - 1; i++) {
            y = (state->key[i] & UPPER_MASK) | (state->key[i + 1] & LOWER_MASK);
            state->key[i] = state->key[i + (M - N)] ^ (y >> 1) ^ (-(y & 1) & MATRIX_A);
        }
        y = (state->key[N - 1] & UPPER_MASK) | (state->key[0] & LOWER_MASK);
        state->key[N - 1] = state->key[M - 1] ^ (y >> 1) ^ (-(y & 1) & MATRIX_A);

        state->pos = 0;
    }

    y = state->key[state->pos++];

    /* Tempering */
    y ^= (y >> 11);
    y ^= (y << 7) & 0x9d2c5680UL;
    y ^= (y << 15) & 0xefc60000UL;
    y ^= (y >> 18);

    return y;
}

inline double rk_double(rk_state *state)
{
    /* shifts : 67108864 = 0x4000000, 9007199254740992 = 0x20000000000000 */
    long a = rk_random(state) >> 5, b = rk_random(state) >> 6;
    return (a * 67108864.0 + b) / 9007199254740992.0;
}

// Initialise the random generator with the specified seed
void rseed(unsigned long seed)
{
    rk_seed(seed, &localState);
    //srand(seed);
}

unsigned long rndl()
{
    return rk_random(&localState);
}

// Returns a uniform distributed double value in the interval ]0,1[
double rnd()
{
    return rk_double(&localState);
}
/*
 * This file is part of the VanitySearch distribution (https://github.com/JeanLucPons/VanitySearch).
 * Copyright (c) 2019 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#ifndef RANDOM_H
#define RANDOM_H

double rnd();
unsigned long rndl();
void rseed(unsigned long seed);

#endif
/*
 * This file is part of the VanitySearch distribution (https://github.com/JeanLucPons/VanitySearch).
 * Copyright (c) 2019 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#include "SECP256k1.h"
#include "hash/sha256.h"
#include "hash/ripemd160.h"
#include "Base58.h"
#include "Bech32.h"
#include <string.h>

Secp256K1::Secp256K1()
{
}

void Secp256K1::Init()
{

    // Prime for the finite field
    Int P;
    P.SetBase16("FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F");

    // Set up field
    Int::SetupField(&P);

    // Generator point and order
    G.x.SetBase16("79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798");
    G.y.SetBase16("483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8");
    G.z.SetInt32(1);
    order.SetBase16("FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141");

    Int::InitK1(&order);

    // Compute Generator table
    Point N(G);
    for (int i = 0; i < 32; i++) {
        GTable[i * 256] = N;
        N = DoubleDirect(N);
        for (int j = 1; j < 255; j++) {
            GTable[i * 256 + j] = N;
            N = AddDirect(N, GTable[i * 256]);
        }
        GTable[i * 256 + 255] = N; // Dummy point for check function
    }

}

Secp256K1::~Secp256K1()
{
}

void PrintResult(bool ok)
{
    if (ok) {
        printf("OK\n");
    } else {
        printf("Failed !\n");
    }
}

void CheckAddress(Secp256K1 *T, std::string address, std::string privKeyStr)
{

    bool isCompressed;
    int type;

    Int privKey = T->DecodePrivateKey((char *)privKeyStr.c_str(), &isCompressed);
    Point pub = T->ComputePublicKey(&privKey);

    switch (address.data()[0]) {
    case '1':
        type = P2PKH; break;
    case '3':
        type = P2SH; break;
    case 'b':
    case 'B':
        type = BECH32; break;
    default:
        printf("Failed ! \n%s Address format not supported\n", address.c_str());
        return;
    }

    std::string calcAddress = T->GetAddress(type, isCompressed, pub);

    printf("Adress : %s ", address.c_str());

    if (address == calcAddress) {
        printf("OK!\n");
        return;
    }

    printf("Failed ! \n %s\n", calcAddress.c_str());

}

void Secp256K1::Check()
{

    printf("Check Generator :");

    bool ok = true;
    int i = 0;
    while (i < 256 * 32 && EC(GTable[i])) {
        i++;
    }
    PrintResult(i == 256 * 32);

    printf("Check Double :");
    Point Pt(G);
    Point R1;
    Point R2;
    Point R3;
    R1 = Double(G);
    R1.Reduce();
    PrintResult(EC(R1));

    printf("Check Add :");
    R2 = Add(G, R1);
    R3 = Add(R1, R2);
    R3.Reduce();
    PrintResult(EC(R3));

    printf("Check GenKey :");
    Int privKey;
    privKey.SetBase16("46b9e861b63d3509c88b7817275a30d22d62c8cd8fa6486ddee35ef0d8e0495f");
    Point pub = ComputePublicKey(&privKey);
    Point expectedPubKey;
    expectedPubKey.x.SetBase16("2500e7f3fbddf2842903f544ddc87494ce95029ace4e257d54ba77f2bc1f3a88");
    expectedPubKey.y.SetBase16("37a9461c4f1c57fecc499753381e772a128a5820a924a2fa05162eb662987a9f");
    expectedPubKey.z.SetInt32(1);

    PrintResult(pub.equals(expectedPubKey));

    CheckAddress(this, "15t3Nt1zyMETkHbjJTTshxLnqPzQvAtdCe", "5HqoeNmaz17FwZRqn7kCBP1FyJKSe4tt42XZB7426EJ2MVWDeqk");
    CheckAddress(this, "1BoatSLRHtKNngkdXEeobR76b53LETtpyT", "5J4XJRyLVgzbXEgh8VNi4qovLzxRftzMd8a18KkdXv4EqAwX3tS");
    CheckAddress(this, "1Test6BNjSJC5qwYXsjwKVLvz7DpfLehy", "5HytzR8p5hp8Cfd8jsVFnwMNXMsEW1sssFxMQYqEUjGZN72iLJ2");
    CheckAddress(this, "16S5PAsGZ8VFM1CRGGLqm37XHrp46f6CTn", "KxMUSkFhEzt2eJHscv2vNSTnnV2cgAXgL4WDQBTx7Ubd9TZmACAz");
    CheckAddress(this, "1Tst2RwMxZn9cYY5mQhCdJic3JJrK7Fq7", "L1vamTpSeK9CgynRpSJZeqvUXf6dJa25sfjb2uvtnhj65R5TymgF");
    CheckAddress(this, "3CyQYcByvcWK8BkYJabBS82yDLNWt6rWSx", "KxMUSkFhEzt2eJHscv2vNSTnnV2cgAXgL4WDQBTx7Ubd9TZmACAz");
    CheckAddress(this, "31to1KQe67YjoDfYnwFJThsGeQcFhVDM5Q", "KxV2Tx5jeeqLHZ1V9ufNv1doTZBZuAc5eY24e6b27GTkDhYwVad7");
    CheckAddress(this, "bc1q6tqytpg06uhmtnhn9s4f35gkt8yya5a24dptmn", "L2wAVD273GwAxGuEDHvrCqPfuWg5wWLZWy6H3hjsmhCvNVuCERAQ");

    // 1ViViGLEawN27xRzGrEhhYPQrZiTKvKLo
    pub.x.SetBase16(/*04*/"75249c39f38baa6bf20ab472191292349426dc3652382cdc45f65695946653dc");
    pub.y.SetBase16("978b2659122fe1df1be132167f27b74e5d4a2f3ecbbbd0b3fbcc2f4983518674");
    printf("Check Calc PubKey (full) %s :", GetAddress(P2PKH, false, pub).c_str());
    PrintResult(EC(pub));

    // 385cR5DM96n1HvBDMzLHPYcw89fZAXULJP
    pub.x.SetBase16(/*03*/"c931af9f331b7a9eb2737667880dacb91428906fbffad0173819a873172d21c4");
    pub.y = GetY(pub.x, false);
    printf("Check Calc PubKey (even) %s:", GetAddress(P2SH, true, pub).c_str());
    PrintResult(EC(pub));

    // 18aPiLmTow7Xgu96msrDYvSSWweCvB9oBA
    pub.x.SetBase16(/*03*/"3bf3d80f868fa33c6353012cb427e98b080452f19b5c1149ea2acfe4b7599739");
    pub.y = GetY(pub.x, false);
    printf("Check Calc PubKey (odd) %s:", GetAddress(P2PKH, true, pub).c_str());
    PrintResult(EC(pub));

}


Point Secp256K1::ComputePublicKey(Int *privKey)
{

    int i = 0;
    uint8_t b;
    Point Q;
    Q.Clear();

    // Search first significant byte
    for (i = 0; i < 32; i++) {
        b = privKey->GetByte(i);
        if (b)
            break;
    }
    Q = GTable[256 * i + (b - 1)];
    i++;

    for (; i < 32; i++) {
        b = privKey->GetByte(i);
        if (b)
            Q = Add2(Q, GTable[256 * i + (b - 1)]);
    }

    Q.Reduce();
    return Q;

}

Point Secp256K1::NextKey(Point &key)
{
    // Input key must be reduced and different from G
    // in order to use AddDirect
    return AddDirect(key, G);
}

Int Secp256K1::DecodePrivateKey(char *key, bool *compressed)
{

    Int ret;
    ret.SetInt32(0);
    std::vector<unsigned char> privKey;

    if (key[0] == '5') {

        // Not compressed
        DecodeBase58(key, privKey);
        if (privKey.size() != 37) {
            printf("Invalid private key, size != 37 (size=%d)!\n", (int)privKey.size());
            ret.SetInt32(-1);
            return ret;
        }

        if (privKey[0] != 0x80) {
            printf("Invalid private key, wrong prefix !\n");
            return ret;
        }

        int count = 31;
        for (int i = 1; i < 33; i++)
            ret.SetByte(count--, privKey[i]);

        // Compute checksum
        unsigned char c[4];
        sha256_checksum(privKey.data(), 33, c);

        if (c[0] != privKey[33] || c[1] != privKey[34] ||
            c[2] != privKey[35] || c[3] != privKey[36]) {
            printf("Warning, Invalid private key checksum !\n");
        }

        *compressed = false;
        return ret;

    } else if (key[0] == 'K' || key[0] == 'L') {

        // Compressed
        DecodeBase58(key, privKey);
        if (privKey.size() != 38) {
            printf("Invalid private key, size != 38 (size=%d)!\n", (int)privKey.size());
            ret.SetInt32(-1);
            return ret;
        }

        int count = 31;
        for (int i = 1; i < 33; i++)
            ret.SetByte(count--, privKey[i]);

        // Compute checksum
        unsigned char c[4];
        sha256_checksum(privKey.data(), 34, c);

        if (c[0] != privKey[34] || c[1] != privKey[35] ||
            c[2] != privKey[36] || c[3] != privKey[37]) {
            printf("Warning, Invalid private key checksum !\n");
        }

        *compressed = true;
        return ret;

    }

    printf("Invalid private key, not starting with 5,K or L !\n");
    ret.SetInt32(-1);
    return ret;

}

#define KEYBUFFCOMP(buff,p) \
(buff)[0] = ((p).x.bits[7] >> 8) | ((uint32_t)(0x2 + (p).y.IsOdd()) << 24); \
(buff)[1] = ((p).x.bits[6] >> 8) | ((p).x.bits[7] <<24); \
(buff)[2] = ((p).x.bits[5] >> 8) | ((p).x.bits[6] <<24); \
(buff)[3] = ((p).x.bits[4] >> 8) | ((p).x.bits[5] <<24); \
(buff)[4] = ((p).x.bits[3] >> 8) | ((p).x.bits[4] <<24); \
(buff)[5] = ((p).x.bits[2] >> 8) | ((p).x.bits[3] <<24); \
(buff)[6] = ((p).x.bits[1] >> 8) | ((p).x.bits[2] <<24); \
(buff)[7] = ((p).x.bits[0] >> 8) | ((p).x.bits[1] <<24); \
(buff)[8] = 0x00800000 | ((p).x.bits[0] <<24); \
(buff)[9] = 0; \
(buff)[10] = 0; \
(buff)[11] = 0; \
(buff)[12] = 0; \
(buff)[13] = 0; \
(buff)[14] = 0; \
(buff)[15] = 0x108;

#define KEYBUFFUNCOMP(buff,p) \
(buff)[0] = ((p).x.bits[7] >> 8) | 0x04000000; \
(buff)[1] = ((p).x.bits[6] >> 8) | ((p).x.bits[7] <<24); \
(buff)[2] = ((p).x.bits[5] >> 8) | ((p).x.bits[6] <<24); \
(buff)[3] = ((p).x.bits[4] >> 8) | ((p).x.bits[5] <<24); \
(buff)[4] = ((p).x.bits[3] >> 8) | ((p).x.bits[4] <<24); \
(buff)[5] = ((p).x.bits[2] >> 8) | ((p).x.bits[3] <<24); \
(buff)[6] = ((p).x.bits[1] >> 8) | ((p).x.bits[2] <<24); \
(buff)[7] = ((p).x.bits[0] >> 8) | ((p).x.bits[1] <<24); \
(buff)[8] = ((p).y.bits[7] >> 8) | ((p).x.bits[0] <<24); \
(buff)[9] = ((p).y.bits[6] >> 8) | ((p).y.bits[7] <<24); \
(buff)[10] = ((p).y.bits[5] >> 8) | ((p).y.bits[6] <<24); \
(buff)[11] = ((p).y.bits[4] >> 8) | ((p).y.bits[5] <<24); \
(buff)[12] = ((p).y.bits[3] >> 8) | ((p).y.bits[4] <<24); \
(buff)[13] = ((p).y.bits[2] >> 8) | ((p).y.bits[3] <<24); \
(buff)[14] = ((p).y.bits[1] >> 8) | ((p).y.bits[2] <<24); \
(buff)[15] = ((p).y.bits[0] >> 8) | ((p).y.bits[1] <<24); \
(buff)[16] = 0x00800000 | ((p).y.bits[0] <<24); \
(buff)[17] = 0; \
(buff)[18] = 0; \
(buff)[19] = 0; \
(buff)[20] = 0; \
(buff)[21] = 0; \
(buff)[22] = 0; \
(buff)[23] = 0; \
(buff)[24] = 0; \
(buff)[25] = 0; \
(buff)[26] = 0; \
(buff)[27] = 0; \
(buff)[28] = 0; \
(buff)[29] = 0; \
(buff)[30] = 0; \
(buff)[31] = 0x208;

#define KEYBUFFSCRIPT(buff,h) \
(buff)[0] = 0x00140000 | (uint32_t)h[0] << 8 | (uint32_t)h[1]; \
(buff)[1] = (uint32_t)h[2] << 24 | (uint32_t)h[3] << 16 | (uint32_t)h[4] << 8 | (uint32_t)h[5];\
(buff)[2] = (uint32_t)h[6] << 24 | (uint32_t)h[7] << 16 | (uint32_t)h[8] << 8 | (uint32_t)h[9];\
(buff)[3] = (uint32_t)h[10] << 24 | (uint32_t)h[11] << 16 | (uint32_t)h[12] << 8 | (uint32_t)h[13];\
(buff)[4] = (uint32_t)h[14] << 24 | (uint32_t)h[15] << 16 | (uint32_t)h[16] << 8 | (uint32_t)h[17];\
(buff)[5] = (uint32_t)h[18] << 24 | (uint32_t)h[19] << 16 | 0x8000; \
(buff)[6] = 0; \
(buff)[7] = 0; \
(buff)[8] = 0; \
(buff)[9] = 0; \
(buff)[10] = 0; \
(buff)[11] = 0; \
(buff)[12] = 0; \
(buff)[13] = 0; \
(buff)[14] = 0; \
(buff)[15] = 0xB0;

void Secp256K1::GetHash160(int type, bool compressed,
                           Point &k0, Point &k1, Point &k2, Point &k3,
                           uint8_t *h0, uint8_t *h1, uint8_t *h2, uint8_t *h3)
{

#ifdef WIN64
    __declspec(align(16)) unsigned char sh0[64];
    __declspec(align(16)) unsigned char sh1[64];
    __declspec(align(16)) unsigned char sh2[64];
    __declspec(align(16)) unsigned char sh3[64];
#else
    unsigned char sh0[64] __attribute__((aligned(16)));
    unsigned char sh1[64] __attribute__((aligned(16)));
    unsigned char sh2[64] __attribute__((aligned(16)));
    unsigned char sh3[64] __attribute__((aligned(16)));
#endif

    switch (type) {

    case P2PKH:
    case BECH32: {

        if (!compressed) {

            uint32_t b0[32];
            uint32_t b1[32];
            uint32_t b2[32];
            uint32_t b3[32];

            KEYBUFFUNCOMP(b0, k0);
            KEYBUFFUNCOMP(b1, k1);
            KEYBUFFUNCOMP(b2, k2);
            KEYBUFFUNCOMP(b3, k3);

            sha256sse_2B(b0, b1, b2, b3, sh0, sh1, sh2, sh3);
            ripemd160sse_32(sh0, sh1, sh2, sh3, h0, h1, h2, h3);

        } else {

            uint32_t b0[16];
            uint32_t b1[16];
            uint32_t b2[16];
            uint32_t b3[16];

            KEYBUFFCOMP(b0, k0);
            KEYBUFFCOMP(b1, k1);
            KEYBUFFCOMP(b2, k2);
            KEYBUFFCOMP(b3, k3);

            sha256sse_1B(b0, b1, b2, b3, sh0, sh1, sh2, sh3);
            ripemd160sse_32(sh0, sh1, sh2, sh3, h0, h1, h2, h3);

        }

    }
    break;

    case P2SH: {

        unsigned char kh0[20];
        unsigned char kh1[20];
        unsigned char kh2[20];
        unsigned char kh3[20];

        GetHash160(P2PKH, compressed, k0, k1, k2, k3, kh0, kh1, kh2, kh3);

        // Redeem Script (1 to 1 P2SH)
        uint32_t b0[16];
        uint32_t b1[16];
        uint32_t b2[16];
        uint32_t b3[16];

        KEYBUFFSCRIPT(b0, kh0);
        KEYBUFFSCRIPT(b1, kh1);
        KEYBUFFSCRIPT(b2, kh2);
        KEYBUFFSCRIPT(b3, kh3);

        sha256sse_1B(b0, b1, b2, b3, sh0, sh1, sh2, sh3);
        ripemd160sse_32(sh0, sh1, sh2, sh3, h0, h1, h2, h3);

    }
    break;

    }

}

uint8_t Secp256K1::GetByte(std::string &str, int idx)
{

    char tmp[3];
    int  val;

    tmp[0] = str.data()[2 * idx];
    tmp[1] = str.data()[2 * idx + 1];
    tmp[2] = 0;

    if (sscanf(tmp, "%X", &val) != 1) {
        printf("ParsePublicKeyHex: Error invalid public key specified (unexpected hexadecimal digit)\n");
        exit(-1);
    }

    return (uint8_t)val;

}

Point Secp256K1::ParsePublicKeyHex(std::string str, bool &isCompressed)
{

    Point ret;
    ret.Clear();

    if (str.length() < 2) {
        printf("ParsePublicKeyHex: Error invalid public key specified (66 or 130 character length)\n");
        exit(-1);
    }

    uint8_t type = GetByte(str, 0);

    switch (type) {

    case 0x02:
        if (str.length() != 66) {
            printf("ParsePublicKeyHex: Error invalid public key specified (66 character length)\n");
            exit(-1);
        }
        for (int i = 0; i < 32; i++)
            ret.x.SetByte(31 - i, GetByte(str, i + 1));
        ret.y = GetY(ret.x, true);
        isCompressed = true;
        break;

    case 0x03:
        if (str.length() != 66) {
            printf("ParsePublicKeyHex: Error invalid public key specified (66 character length)\n");
            exit(-1);
        }
        for (int i = 0; i < 32; i++)
            ret.x.SetByte(31 - i, GetByte(str, i + 1));
        ret.y = GetY(ret.x, false);
        isCompressed = true;
        break;

    case 0x04:
        if (str.length() != 130) {
            printf("ParsePublicKeyHex: Error invalid public key specified (130 character length)\n");
            exit(-1);
        }
        for (int i = 0; i < 32; i++)
            ret.x.SetByte(31 - i, GetByte(str, i + 1));
        for (int i = 0; i < 32; i++)
            ret.y.SetByte(31 - i, GetByte(str, i + 33));
        isCompressed = false;
        break;

    default:
        printf("ParsePublicKeyHex: Error invalid public key specified (Unexpected prefix (only 02,03 or 04 allowed)\n");
        exit(-1);
    }

    ret.z.SetInt32(1);

    if (!EC(ret)) {
        printf("ParsePublicKeyHex: Error invalid public key specified (Not lie on elliptic curve)\n");
        exit(-1);
    }

    return ret;

}

std::string Secp256K1::GetPublicKeyHex(bool compressed, Point &pubKey)
{

    unsigned char publicKeyBytes[128];
    char tmp[3];
    std::string ret;

    if (!compressed) {

        // Full public key
        publicKeyBytes[0] = 0x4;
        pubKey.x.Get32Bytes(publicKeyBytes + 1);
        pubKey.y.Get32Bytes(publicKeyBytes + 33);

        for (int i = 0; i < 65; i++) {
            sprintf(tmp, "%02X", (int)publicKeyBytes[i]);
            ret.append(tmp);
        }

    } else {

        // Compressed public key
        publicKeyBytes[0] = pubKey.y.IsEven() ? 0x2 : 0x3;
        pubKey.x.Get32Bytes(publicKeyBytes + 1);

        for (int i = 0; i < 33; i++) {
            sprintf(tmp, "%02X", (int)publicKeyBytes[i]);
            ret.append(tmp);
        }

    }

    return ret;

}

void Secp256K1::GetHash160(int type, bool compressed, Point &pubKey, unsigned char *hash)
{

    unsigned char shapk[64];

    switch (type) {

    case P2PKH:
    case BECH32: {
        unsigned char publicKeyBytes[128];

        if (!compressed) {

            // Full public key
            publicKeyBytes[0] = 0x4;
            pubKey.x.Get32Bytes(publicKeyBytes + 1);
            pubKey.y.Get32Bytes(publicKeyBytes + 33);
            sha256_65(publicKeyBytes, shapk);

        } else {

            // Compressed public key
            publicKeyBytes[0] = pubKey.y.IsEven() ? 0x2 : 0x3;
            pubKey.x.Get32Bytes(publicKeyBytes + 1);
            sha256_33(publicKeyBytes, shapk);

        }

        ripemd160_32(shapk, hash);
    }
    break;

    case P2SH: {

        // Redeem Script (1 to 1 P2SH)
        unsigned char script[64];

        script[0] = 0x00;  // OP_0
        script[1] = 0x14;  // PUSH 20 bytes
        GetHash160(P2PKH, compressed, pubKey, script + 2);

        sha256(script, 22, shapk);
        ripemd160_32(shapk, hash);

    }
    break;

    }

}

std::string Secp256K1::GetPrivAddress(bool compressed, Int &privKey)
{

    unsigned char address[38];

    address[0] = 0x80; // Mainnet
    privKey.Get32Bytes(address + 1);

    if (compressed) {

        // compressed suffix
        address[33] = 1;
        sha256_checksum(address, 34, address + 34);
        return EncodeBase58(address, address + 38);

    } else {

        // Compute checksum
        sha256_checksum(address, 33, address + 33);
        return EncodeBase58(address, address + 37);

    }

}

#define CHECKSUM(buff,A) \
(buff)[0] = (uint32_t)A[0] << 24 | (uint32_t)A[1] << 16 | (uint32_t)A[2] << 8 | (uint32_t)A[3];\
(buff)[1] = (uint32_t)A[4] << 24 | (uint32_t)A[5] << 16 | (uint32_t)A[6] << 8 | (uint32_t)A[7];\
(buff)[2] = (uint32_t)A[8] << 24 | (uint32_t)A[9] << 16 | (uint32_t)A[10] << 8 | (uint32_t)A[11];\
(buff)[3] = (uint32_t)A[12] << 24 | (uint32_t)A[13] << 16 | (uint32_t)A[14] << 8 | (uint32_t)A[15];\
(buff)[4] = (uint32_t)A[16] << 24 | (uint32_t)A[17] << 16 | (uint32_t)A[18] << 8 | (uint32_t)A[19];\
(buff)[5] = (uint32_t)A[20] << 24 | 0x800000;\
(buff)[6] = 0; \
(buff)[7] = 0; \
(buff)[8] = 0; \
(buff)[9] = 0; \
(buff)[10] = 0; \
(buff)[11] = 0; \
(buff)[12] = 0; \
(buff)[13] = 0; \
(buff)[14] = 0; \
(buff)[15] = 0xA8;

std::vector<std::string> Secp256K1::GetAddress(int type, bool compressed, unsigned char *h1, unsigned char *h2, unsigned char *h3, unsigned char *h4)
{

    std::vector<std::string> ret;

    unsigned char add1[25];
    unsigned char add2[25];
    unsigned char add3[25];
    unsigned char add4[25];
    uint32_t b1[16];
    uint32_t b2[16];
    uint32_t b3[16];
    uint32_t b4[16];

    switch (type) {

    case P2PKH:
        add1[0] = 0x00;
        add2[0] = 0x00;
        add3[0] = 0x00;
        add4[0] = 0x00;
        break;

    case P2SH:
        add1[0] = 0x05;
        add2[0] = 0x05;
        add3[0] = 0x05;
        add4[0] = 0x05;
        break;

    case BECH32: {
        char output[128];
        segwit_addr_encode(output, "bc", 0, h1, 20);
        ret.push_back(std::string(output));
        segwit_addr_encode(output, "bc", 0, h2, 20);
        ret.push_back(std::string(output));
        segwit_addr_encode(output, "bc", 0, h3, 20);
        ret.push_back(std::string(output));
        segwit_addr_encode(output, "bc", 0, h4, 20);
        ret.push_back(std::string(output));
        return ret;
    }
    break;
    }

    memcpy(add1 + 1, h1, 20);
    memcpy(add2 + 1, h2, 20);
    memcpy(add3 + 1, h3, 20);
    memcpy(add4 + 1, h4, 20);
    CHECKSUM(b1, add1);
    CHECKSUM(b2, add2);
    CHECKSUM(b3, add3);
    CHECKSUM(b4, add4);
    sha256sse_checksum(b1, b2, b3, b4, add1 + 21, add2 + 21, add3 + 21, add4 + 21);

    // Base58
    ret.push_back(EncodeBase58(add1, add1 + 25));
    ret.push_back(EncodeBase58(add2, add2 + 25));
    ret.push_back(EncodeBase58(add3, add3 + 25));
    ret.push_back(EncodeBase58(add4, add4 + 25));

    return ret;

}

std::string Secp256K1::GetAddress(int type, bool compressed, unsigned char *hash160)
{

    unsigned char address[25];
    switch (type) {

    case P2PKH:
        address[0] = 0x00;
        break;

    case P2SH:
        address[0] = 0x05;
        break;

    case BECH32: {
        char output[128];
        segwit_addr_encode(output, "bc", 0, hash160, 20);
        return std::string(output);
    }
    break;
    }
    memcpy(address + 1, hash160, 20);
    sha256_checksum(address, 21, address + 21);

    // Base58
    return EncodeBase58(address, address + 25);

}

std::string Secp256K1::GetAddress(int type, bool compressed, Point &pubKey)
{

    unsigned char address[25];

    switch (type) {

    case P2PKH:
        address[0] = 0x00;
        break;

    case BECH32: {
        if (!compressed) {
            return " BECH32: Only compressed key ";
        }
        char output[128];
        uint8_t h160[20];
        GetHash160(type, compressed, pubKey, h160);
        segwit_addr_encode(output, "bc", 0, h160, 20);
        return std::string(output);
    }
    break;

    case P2SH:
        if (!compressed) {
            return " P2SH: Only compressed key ";
        }
        address[0] = 0x05;
        break;
    }

    GetHash160(type, compressed, pubKey, address + 1);
    sha256_checksum(address, 21, address + 21);

    // Base58
    return EncodeBase58(address, address + 25);

}

bool Secp256K1::CheckPudAddress(std::string address)
{

    std::vector<unsigned char> pubKey;
    DecodeBase58(address, pubKey);

    if (pubKey.size() != 25)
        return false;

    // Check checksum
    unsigned char chk[4];
    sha256_checksum(pubKey.data(), 21, chk);

    return (pubKey[21] == chk[0]) &&
           (pubKey[22] == chk[1]) &&
           (pubKey[23] == chk[2]) &&
           (pubKey[24] == chk[3]);

}

Point Secp256K1::AddDirect(Point &p1, Point &p2)
{

    Int _s;
    Int _p;
    Int dy;
    Int dx;
    Point r;
    r.z.SetInt32(1);

    dy.ModSub(&p2.y, &p1.y);
    dx.ModSub(&p2.x, &p1.x);
    dx.ModInv();
    _s.ModMulK1(&dy, &dx);    // s = (p2.y-p1.y)*inverse(p2.x-p1.x);

    _p.ModSquareK1(&_s);       // _p = pow2(s)

    r.x.ModSub(&_p, &p1.x);
    r.x.ModSub(&p2.x);       // rx = pow2(s) - p1.x - p2.x;

    r.y.ModSub(&p2.x, &r.x);
    r.y.ModMulK1(&_s);
    r.y.ModSub(&p2.y);       // ry = - p2.y - s*(ret.x-p2.x);

    return r;

}

Point Secp256K1::Add2(Point &p1, Point &p2)
{

    // P2.z = 1

    Int u;
    Int v;
    Int u1;
    Int v1;
    Int vs2;
    Int vs3;
    Int us2;
    Int a;
    Int us2w;
    Int vs2v2;
    Int vs3u2;
    Int _2vs2v2;
    Point r;

    u1.ModMulK1(&p2.y, &p1.z);
    v1.ModMulK1(&p2.x, &p1.z);
    u.ModSub(&u1, &p1.y);
    v.ModSub(&v1, &p1.x);
    us2.ModSquareK1(&u);
    vs2.ModSquareK1(&v);
    vs3.ModMulK1(&vs2, &v);
    us2w.ModMulK1(&us2, &p1.z);
    vs2v2.ModMulK1(&vs2, &p1.x);
    _2vs2v2.ModAdd(&vs2v2, &vs2v2);
    a.ModSub(&us2w, &vs3);
    a.ModSub(&_2vs2v2);

    r.x.ModMulK1(&v, &a);

    vs3u2.ModMulK1(&vs3, &p1.y);
    r.y.ModSub(&vs2v2, &a);
    r.y.ModMulK1(&r.y, &u);
    r.y.ModSub(&vs3u2);

    r.z.ModMulK1(&vs3, &p1.z);

    return r;

}

Point Secp256K1::Add(Point &p1, Point &p2)
{

    Int u;
    Int v;
    Int u1;
    Int u2;
    Int v1;
    Int v2;
    Int vs2;
    Int vs3;
    Int us2;
    Int w;
    Int a;
    Int us2w;
    Int vs2v2;
    Int vs3u2;
    Int _2vs2v2;
    Int x3;
    Int vs3y1;
    Point r;

    /*
    U1 = Y2 * Z1
    U2 = Y1 * Z2
    V1 = X2 * Z1
    V2 = X1 * Z2
    if (V1 == V2)
      if (U1 != U2)
        return POINT_AT_INFINITY
      else
        return POINT_DOUBLE(X1, Y1, Z1)
    U = U1 - U2
    V = V1 - V2
    W = Z1 * Z2
    A = U ^ 2 * W - V ^ 3 - 2 * V ^ 2 * V2
    X3 = V * A
    Y3 = U * (V ^ 2 * V2 - A) - V ^ 3 * U2
    Z3 = V ^ 3 * W
    return (X3, Y3, Z3)
    */

    u1.ModMulK1(&p2.y, &p1.z);
    u2.ModMulK1(&p1.y, &p2.z);
    v1.ModMulK1(&p2.x, &p1.z);
    v2.ModMulK1(&p1.x, &p2.z);
    u.ModSub(&u1, &u2);
    v.ModSub(&v1, &v2);
    w.ModMulK1(&p1.z, &p2.z);
    us2.ModSquareK1(&u);
    vs2.ModSquareK1(&v);
    vs3.ModMulK1(&vs2, &v);
    us2w.ModMulK1(&us2, &w);
    vs2v2.ModMulK1(&vs2, &v2);
    _2vs2v2.ModAdd(&vs2v2, &vs2v2);
    a.ModSub(&us2w, &vs3);
    a.ModSub(&_2vs2v2);

    r.x.ModMulK1(&v, &a);

    vs3u2.ModMulK1(&vs3, &u2);
    r.y.ModSub(&vs2v2, &a);
    r.y.ModMulK1(&r.y, &u);
    r.y.ModSub(&vs3u2);

    r.z.ModMulK1(&vs3, &w);

    return r;
}

Point Secp256K1::DoubleDirect(Point &p)
{

    Int _s;
    Int _p;
    Int a;
    Point r;
    r.z.SetInt32(1);

    _s.ModMulK1(&p.x, &p.x);
    _p.ModAdd(&_s, &_s);
    _p.ModAdd(&_s);

    a.ModAdd(&p.y, &p.y);
    a.ModInv();
    _s.ModMulK1(&_p, &a);    // s = (3*pow2(p.x))*inverse(2*p.y);

    _p.ModMulK1(&_s, &_s);
    a.ModAdd(&p.x, &p.x);
    a.ModNeg();
    r.x.ModAdd(&a, &_p);   // rx = pow2(s) + neg(2*p.x);

    a.ModSub(&r.x, &p.x);

    _p.ModMulK1(&a, &_s);
    r.y.ModAdd(&_p, &p.y);
    r.y.ModNeg();           // ry = neg(p.y + s*(ret.x+neg(p.x)));

    return r;
}

Point Secp256K1::Double(Point &p)
{


    /*
    if (Y == 0)
      return POINT_AT_INFINITY
      W = a * Z ^ 2 + 3 * X ^ 2
      S = Y * Z
      B = X * Y*S
      H = W ^ 2 - 8 * B
      X' = 2*H*S
      Y' = W*(4*B - H) - 8*Y^2*S^2
      Z' = 8*S^3
      return (X', Y', Z')
    */

    Int z2;
    Int x2;
    Int _3x2;
    Int w;
    Int s;
    Int s2;
    Int b;
    Int _8b;
    Int _8y2s2;
    Int y2;
    Int h;
    Point r;

    z2.ModSquareK1(&p.z);
    z2.SetInt32(0); // a=0
    x2.ModSquareK1(&p.x);
    _3x2.ModAdd(&x2, &x2);
    _3x2.ModAdd(&x2);
    w.ModAdd(&z2, &_3x2);
    s.ModMulK1(&p.y, &p.z);
    b.ModMulK1(&p.y, &s);
    b.ModMulK1(&p.x);
    h.ModSquareK1(&w);
    _8b.ModAdd(&b, &b);
    _8b.ModDouble();
    _8b.ModDouble();
    h.ModSub(&_8b);

    r.x.ModMulK1(&h, &s);
    r.x.ModAdd(&r.x);

    s2.ModSquareK1(&s);
    y2.ModSquareK1(&p.y);
    _8y2s2.ModMulK1(&y2, &s2);
    _8y2s2.ModDouble();
    _8y2s2.ModDouble();
    _8y2s2.ModDouble();

    r.y.ModAdd(&b, &b);
    r.y.ModAdd(&r.y, &r.y);
    r.y.ModSub(&h);
    r.y.ModMulK1(&w);
    r.y.ModSub(&_8y2s2);

    r.z.ModMulK1(&s2, &s);
    r.z.ModDouble();
    r.z.ModDouble();
    r.z.ModDouble();

    return r;
}

Int Secp256K1::GetY(Int x, bool isEven)
{

    Int _s;
    Int _p;

    _s.ModSquareK1(&x);
    _p.ModMulK1(&_s, &x);
    _p.ModAdd(7);
    _p.ModSqrt();

    if (!_p.IsEven() && isEven) {
        _p.ModNeg();
    } else if (_p.IsEven() && !isEven) {
        _p.ModNeg();
    }

    return _p;

}

bool Secp256K1::EC(Point &p)
{

    Int _s;
    Int _p;

    _s.ModSquareK1(&p.x);
    _p.ModMulK1(&_s, &p.x);
    _p.ModAdd(7);
    _s.ModMulK1(&p.y, &p.y);
    _s.ModSub(&_p);

    return _s.IsZero(); // ( ((pow2(y) - (pow3(x) + 7)) % P) == 0 );

}
/*
 * This file is part of the VanitySearch distribution (https://github.com/JeanLucPons/VanitySearch).
 * Copyright (c) 2019 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#ifndef SECP256K1H
#define SECP256K1H

#include "Point.h"
#include <string>
#include <vector>

// Address type
#define P2PKH  0
#define P2SH   1
#define BECH32 2

class Secp256K1
{

public:

    Secp256K1();
    ~Secp256K1();
    void Init();
    Point ComputePublicKey(Int *privKey);
    Point NextKey(Point &key);
    void Check();
    bool  EC(Point &p);

    void GetHash160(int type, bool compressed,
                    Point &k0, Point &k1, Point &k2, Point &k3,
                    uint8_t *h0, uint8_t *h1, uint8_t *h2, uint8_t *h3);

    void GetHash160(int type, bool compressed, Point &pubKey, unsigned char *hash);

    std::string GetAddress(int type, bool compressed, Point &pubKey);
    std::string GetAddress(int type, bool compressed, unsigned char *hash160);
    std::vector<std::string> GetAddress(int type, bool compressed, unsigned char *h1, unsigned char *h2, unsigned char *h3, unsigned char *h4);
    std::string GetPrivAddress(bool compressed, Int &privKey);
    std::string GetPublicKeyHex(bool compressed, Point &p);
    Point ParsePublicKeyHex(std::string str, bool &isCompressed);

    bool CheckPudAddress(std::string address);

    static Int DecodePrivateKey(char *key, bool *compressed);

    Point Add(Point &p1, Point &p2);
    Point Add2(Point &p1, Point &p2);
    Point AddDirect(Point &p1, Point &p2);
    Point Double(Point &p);
    Point DoubleDirect(Point &p);

    Point G;                 // Generator
    Int   order;             // Curve order

private:

    uint8_t GetByte(std::string &str, int idx);

    Int GetY(Int x, bool isEven);
    Point GTable[256 * 32];     // Generator table

};

#endif // SECP256K1H
/*
 * This file is part of the VanitySearch distribution (https://github.com/JeanLucPons/VanitySearch).
 * Copyright (c) 2019 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#include "Timer.h"

static const char *prefix[] = { "", "Kilo", "Mega", "Giga", "Tera", "Peta", "Hexa" };

#ifdef WIN64

LARGE_INTEGER Timer::perfTickStart;
double Timer::perfTicksPerSec;
LARGE_INTEGER Timer::qwTicksPerSec;
#include <wincrypt.h>

#else

#include <sys/time.h>
#include <unistd.h>
#include <string.h>
time_t Timer::tickStart;

#endif

void Timer::Init()
{

#ifdef WIN64
    QueryPerformanceFrequency(&qwTicksPerSec);
    QueryPerformanceCounter(&perfTickStart);
    perfTicksPerSec = (double)qwTicksPerSec.QuadPart;
#else
    tickStart = time(NULL);
#endif

}

double Timer::get_tick()
{

#ifdef WIN64
    LARGE_INTEGER t, dt;
    QueryPerformanceCounter(&t);
    dt.QuadPart = t.QuadPart - perfTickStart.QuadPart;
    return (double)(dt.QuadPart) / perfTicksPerSec;
#else
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return (double)(tv.tv_sec - tickStart) + (double)tv.tv_usec / 1e6;
#endif

}

std::string Timer::getSeed(int size)
{

    std::string ret;
    char tmp[3];
    unsigned char *buff = (unsigned char *)malloc(size);

#ifdef WIN64

    HCRYPTPROV   hCryptProv = NULL;
    LPCSTR UserName = "KeyContainer";

    if (!CryptAcquireContext(
                &hCryptProv,               // handle to the CSP
                UserName,                  // container name
                NULL,                      // use the default provider
                PROV_RSA_FULL,             // provider type
                0)) {                      // flag values
        //-------------------------------------------------------------------
        // An error occurred in acquiring the context. This could mean
        // that the key container requested does not exist. In this case,
        // the function can be called again to attempt to create a new key
        // container. Error codes are defined in Winerror.h.
        if (GetLastError() == NTE_BAD_KEYSET) {
            if (!CryptAcquireContext(
                        &hCryptProv,
                        UserName,
                        NULL,
                        PROV_RSA_FULL,
                        CRYPT_NEWKEYSET)) {
                printf("CryptAcquireContext(): Could not create a new key container.\n");
                exit(1);
            }
        } else {
            printf("CryptAcquireContext(): A cryptographic service handle could not be acquired.\n");
            exit(1);
        }
    }

    if (!CryptGenRandom(hCryptProv, size, buff)) {
        printf("CryptGenRandom(): Error during random sequence acquisition.\n");
        exit(1);
    }

    CryptReleaseContext(hCryptProv, 0);

#else

    FILE *f = fopen("/dev/urandom", "rb");
    if (f == NULL) {
        printf("Failed to open /dev/urandom %s\n", strerror(errno));
        exit(1);
    }
    if (fread(buff, 1, size, f) != size) {
        printf("Failed to read from /dev/urandom %s\n", strerror(errno));
        exit(1);
    }
    fclose(f);

#endif

    for (int i = 0; i < size; i++) {
        sprintf(tmp, "%02X", buff[i]);
        ret.append(tmp);
    }

    free(buff);
    return ret;

}

uint32_t Timer::getSeed32()
{
    return ::strtoul(getSeed(4).c_str(), NULL, 16);
}

std::string Timer::getResult(const char *unit, int nbTry, double t0, double t1)
{

    char tmp[256];
    int pIdx = 0;
    double nbCallPerSec = (double)nbTry / (t1 - t0);
    while (nbCallPerSec > 1000.0 && pIdx < 5) {
        pIdx++;
        nbCallPerSec = nbCallPerSec / 1000.0;
    }
    sprintf(tmp, "%.3f %s%s/sec", nbCallPerSec, prefix[pIdx], unit);
    return std::string(tmp);

}

void Timer::printResult(const char *unit, int nbTry, double t0, double t1)
{

    printf("%s\n", getResult(unit, nbTry, t0, t1).c_str());

}

int Timer::getCoreNumber()
{

#ifdef WIN64
    SYSTEM_INFO sysinfo;
    GetSystemInfo(&sysinfo);
    return sysinfo.dwNumberOfProcessors;
#else
    // TODO
    return 1;
#endif

}

void Timer::SleepMillis(uint32_t millis)
{

#ifdef WIN64
    Sleep(millis);
#else
    usleep(millis * 1000);
#endif

}
/*
 * This file is part of the VanitySearch distribution (https://github.com/JeanLucPons/VanitySearch).
 * Copyright (c) 2019 Jean Luc PONS.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, version 3.
 *
 * This program is distributed in the hope that it will be useful, but
 * WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
 * General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
*/

#ifndef TIMERH
#define TIMERH

#include <time.h>
#include <string>
#ifdef WIN64
#include <windows.h>
#endif

class Timer
{

public:
    static void Init();
    static double get_tick();
    static void printResult(const char *unit, int nbTry, double t0, double t1);
    static std::string getResult(const char *unit, int nbTry, double t0, double t1);
    static int getCoreNumber();
    static std::string getSeed(int size);
    static uint32_t getSeed32();
    static void SleepMillis(uint32_t millis);

#ifdef WIN64
    static LARGE_INTEGER perfTickStart;
    static double perfTicksPerSec;
    static LARGE_INTEGER qwTicksPerSec;
#else
    static time_t tickStart;
#endif

};

#endif // TIMERH
#include <cstring>
#include <fstream>
#include "sha256.h"

const unsigned int SHA256::sha256_k[64] = //UL = uint32
{
 0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5,
 0x3956c25b, 0x59f111f1, 0x923f82a4, 0xab1c5ed5,
 0xd807aa98, 0x12835b01, 0x243185be, 0x550c7dc3,
 0x72be5d74, 0x80deb1fe, 0x9bdc06a7, 0xc19bf174,
 0xe49b69c1, 0xefbe4786, 0x0fc19dc6, 0x240ca1cc,
 0x2de92c6f, 0x4a7484aa, 0x5cb0a9dc, 0x76f988da,
 0x983e5152, 0xa831c66d, 0xb00327c8, 0xbf597fc7,
 0xc6e00bf3, 0xd5a79147, 0x06ca6351, 0x14292967,
 0x27b70a85, 0x2e1b2138, 0x4d2c6dfc, 0x53380d13,
 0x650a7354, 0x766a0abb, 0x81c2c92e, 0x92722c85,
 0xa2bfe8a1, 0xa81a664b, 0xc24b8b70, 0xc76c51a3,
 0xd192e819, 0xd6990624, 0xf40e3585, 0x106aa070,
 0x19a4c116, 0x1e376c08, 0x2748774c, 0x34b0bcb5,
 0x391c0cb3, 0x4ed8aa4a, 0x5b9cca4f, 0x682e6ff3,
 0x748f82ee, 0x78a5636f, 0x84c87814, 0x8cc70208,
 0x90befffa, 0xa4506ceb, 0xbef9a3f7, 0xc67178f2 };

void SHA256::transform(const unsigned char* message, unsigned int block_nb){
    uint32 w[64];
    uint32 wv[8];
    uint32 t1, t2;
    const unsigned char* sub_block;
    int i;
    int j;
    for (i = 0; i < (int)block_nb; i++) {
        sub_block = message + (i << 6);
        for (j = 0; j < 16; j++) {
            SHA2_PACK32(&sub_block[j << 2], &w[j]);
        }
        for (j = 16; j < 64; j++) {
            w[j] = SHA256_F4(w[j - 2]) + w[j - 7] + SHA256_F3(w[j - 15]) + w[j - 16];
        }
        for (j = 0; j < 8; j++) {
            wv[j] = m_h[j];
        }
        for (j = 0; j < 64; j++) {
            t1 = wv[7] + SHA256_F2(wv[4]) + SHA2_CH(wv[4], wv[5], wv[6])
                + sha256_k[j] + w[j];
            t2 = SHA256_F1(wv[0]) + SHA2_MAJ(wv[0], wv[1], wv[2]);
            wv[7] = wv[6];
            wv[6] = wv[5];
            wv[5] = wv[4];
            wv[4] = wv[3] + t1;
            wv[3] = wv[2];
            wv[2] = wv[1];
            wv[1] = wv[0];
            wv[0] = t1 + t2;
        }
        for (j = 0; j < 8; j++) {
            m_h[j] += wv[j];
        }
    }
}

void SHA256::init(){
    m_h[0] = 0x6a09e667;
    m_h[1] = 0xbb67ae85;
    m_h[2] = 0x3c6ef372;
    m_h[3] = 0xa54ff53a;
    m_h[4] = 0x510e527f;
    m_h[5] = 0x9b05688c;
    m_h[6] = 0x1f83d9ab;
    m_h[7] = 0x5be0cd19;
    m_len = 0;
    m_tot_len = 0;
}

void SHA256::update(const unsigned char* message, unsigned int len){
    unsigned int block_nb;
    unsigned int new_len, rem_len, tmp_len;
    const unsigned char* shifted_message;
    tmp_len = SHA224_256_BLOCK_SIZE - m_len;
    rem_len = len < tmp_len ? len : tmp_len;
    memcpy(&m_block[m_len], message, rem_len);
    if (m_len + len < SHA224_256_BLOCK_SIZE) {
        m_len += len;
        return;
    }
    new_len = len - rem_len;
    block_nb = new_len / SHA224_256_BLOCK_SIZE;
    shifted_message = message + rem_len;
    transform(m_block, 1);
    transform(shifted_message, block_nb);
    rem_len = new_len % SHA224_256_BLOCK_SIZE;
    memcpy(m_block, &shifted_message[block_nb << 6], rem_len);
    m_len = rem_len;
    m_tot_len += (block_nb + 1) << 6;
}

void SHA256::final(unsigned char* digest){
    unsigned int block_nb;
    unsigned int pm_len;
    unsigned int len_b;
    int i;
    block_nb = (1 + ((SHA224_256_BLOCK_SIZE - 9)
        < (m_len % SHA224_256_BLOCK_SIZE)));
    len_b = (m_tot_len + m_len) << 3;
    pm_len = block_nb << 6;
    memset(m_block + m_len, 0, pm_len - m_len);
    m_block[m_len] = 0x80;
    SHA2_UNPACK32(len_b, m_block + pm_len - 4);
    transform(m_block, block_nb);
    for (i = 0; i < 8; i++) {
        SHA2_UNPACK32(m_h[i], &digest[i << 2]);
    }
}

std::string sha256(std::string input){
    unsigned char digest[SHA256::DIGEST_SIZE];
    memset(digest, 0, SHA256::DIGEST_SIZE);

    SHA256 ctx = SHA256();
    ctx.init();
    ctx.update((unsigned char*)input.c_str(), input.length());
    ctx.final(digest);

    char buf[2 * SHA256::DIGEST_SIZE + 1];
    buf[2 * SHA256::DIGEST_SIZE] = 0;
    for (int i = 0; i < SHA256::DIGEST_SIZE; i++)
        sprintf(buf + i * 2, "%02x", digest[i]);
    return std::string(buf);
}
#ifndef SHA256_H
#define SHA256_H
#include <string>

class SHA256{
protected:
    typedef unsigned char uint8;
    typedef unsigned int uint32;
    typedef unsigned long long uint64;

    const static uint32 sha256_k[];
    static const unsigned int SHA224_256_BLOCK_SIZE = (512 / 8);
public:
    void init();
    void update(const unsigned char* message, unsigned int len);
    void final(unsigned char* digest);
    static const unsigned int DIGEST_SIZE = (256 / 8);

protected:
    void transform(const unsigned char* message, unsigned int block_nb);
    unsigned int m_tot_len;
    unsigned int m_len;
    unsigned char m_block[2 * SHA224_256_BLOCK_SIZE];
    uint32 m_h[8];
};

std::string sha256(std::string input);

#define SHA2_SHFR(x, n)    (x >> n)
#define SHA2_ROTR(x, n)   ((x >> n) | (x << ((sizeof(x) << 3) - n)))
#define SHA2_ROTL(x, n)   ((x << n) | (x >> ((sizeof(x) << 3) - n)))
#define SHA2_CH(x, y, z)  ((x & y) ^ (~x & z))
#define SHA2_MAJ(x, y, z) ((x & y) ^ (x & z) ^ (y & z))
#define SHA256_F1(x) (SHA2_ROTR(x,  2) ^ SHA2_ROTR(x, 13) ^ SHA2_ROTR(x, 22))
#define SHA256_F2(x) (SHA2_ROTR(x,  6) ^ SHA2_ROTR(x, 11) ^ SHA2_ROTR(x, 25))
#define SHA256_F3(x) (SHA2_ROTR(x,  7) ^ SHA2_ROTR(x, 18) ^ SHA2_SHFR(x,  3))
#define SHA256_F4(x) (SHA2_ROTR(x, 17) ^ SHA2_ROTR(x, 19) ^ SHA2_SHFR(x, 10))
#define SHA2_UNPACK32(x, str)                 \
{                                             \
    *((str) + 3) = (uint8) ((x)      );       \
    *((str) + 2) = (uint8) ((x) >>  8);       \
    *((str) + 1) = (uint8) ((x) >> 16);       \
    *((str) + 0) = (uint8) ((x) >> 24);       \
}
#define SHA2_PACK32(str, x)                   \
{                                             \
    *(x) =   ((uint32) *((str) + 3)      )    \
           | ((uint32) *((str) + 2) <<  8)    \
           | ((uint32) *((str) + 1) << 16)    \
           | ((uint32) *((str) + 0) << 24);   \
}
#endif
