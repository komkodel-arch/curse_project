# MAIN CODE

```
#include <iostream>
#include <conio.h>
#include <random>
#include <Windows.h>
#include <sstream>
#include <stdlib.h>
#include <vector>
#include <math.h>
#include <chrono>

using namespace std;

struct genome_setup
{
	short
		production = 121,
		consuming = 134,
		cannibalism = 30,
		fertility = 120,

		energy_limit = 200, // 100 1000
		energy_aware = 60, // ?

		connecting = 10,

		age_limit = 20, // 5 100
		age_breed = 5 // 0, 90
		;
};

struct behavior_adult
{
	float
		move = 10.0f,
		breed = 20.0f,
		consume = 30.0f,
		product = 30.0f,
		share = 5.0f,
		save = 5.0f
		;
};

struct behavior_child
{
	float
		move = 20.0f,
		consume = 35.0f,
		product = 35.0f,
		share = 5.0f,
		save = 5.0f
		;
};

struct cell
{
	genome_setup genome;
	behavior_child bh_child;
	behavior_adult bh_adult;

	short x, y,
		energy = 100,
		age = -1;
	bool alive = true;
};

class Ecosystem
{
private:
	short size_x = 200, size_y = 100;

	vector<cell> cells;


	short dir_x[4] = { -1, 1, 0, 0 };
	short dir_y[4] = { 0, 0, 1, -1 };

	mt19937 gen;
	uniform_int_distribution<> rad{ -2, 2 };
	uniform_int_distribution<> rng{ 1, 100 };
	uniform_int_distribution<> choose_dir{ 1, 12 };

	short radiance()
	{
		return rad(gen);
	}

	bool check(float checked)
	{
		short res = rng(gen);

		if (res <= checked)
			return true;
		else
			return false;
	}

public:

	cell Adam;

	short** map;

	Ecosystem() : gen(chrono::steady_clock::now().time_since_epoch().count())
	{

		cells.reserve(40000);

		map = new short* [size_y];
		for (short y = 0; y < size_y; y++) {
			map[y] = new short[size_x];
			for (short x = 0; x < size_x; x++) {
				map[y][x] = -1;
			}
		}

		for (short x = 12; x < 24; x++) {
			for (short y = 12; y < 24; y++) {
				Adam.x = x; Adam.y = y;
				map[Adam.y][Adam.x] = cells.size();
				cells.push_back(Adam);
			}
		}
	}

	void cellkill(cell& target)
	{
		map[target.y][target.x] = -1;
		target.alive = false;
	}

	bool same(cell& host, cell& target)
	{
		short dif = 0;

		dif += abs(host.genome.age_breed - target.genome.age_breed);
		dif += abs(host.genome.age_limit - target.genome.age_limit);
		dif += abs(host.genome.fertility - target.genome.fertility);
		dif += abs(host.genome.consuming - target.genome.consuming);
		dif += abs(host.genome.production - target.genome.production);
		dif += abs(host.genome.cannibalism - target.genome.cannibalism);
		dif += abs(host.genome.connecting - target.genome.connecting);
		dif += abs(host.genome.energy_limit - target.genome.energy_limit);
		dif += abs(host.genome.energy_aware - target.genome.energy_aware);

		if (dif > 30)
			return true;
		else
			return false;
	}

	bool move(short host_id)
	{
		cell& host = cells[host_id];

		vector<short>avdir;

		for (short i = 0; i <= 3; i++) {
			if (host.x + dir_x[i] >= 0 && host.x + dir_x[i] < size_x &&
				host.y + dir_y[i] >= 0 && host.y + dir_y[i] < size_y &&
				map[host.y + dir_y[i]][host.x + dir_x[i]] == -1)
				avdir.push_back(i);
		}

		if (avdir.size() != 0) {
			short cash;

			short dir_i = choose_dir(gen);
			dir_i = dir_i % avdir.size();

			cash = map[host.y][host.x];
			map[host.y][host.x] = -1;

			host.y += dir_y[avdir[dir_i]]; host.x += dir_x[avdir[dir_i]];
			map[host.y][host.x] = cash;

			host.energy -= 10;

			return true;
		}
		else return false;
	}

	bool consume(short host_id)
	{
		cell& host = cells[host_id];

		vector<short>avdir;


		for (short i = 0; i <= 3; i++) {
			if (host.x + dir_x[i] >= 0 && host.x + dir_x[i] < size_x &&
				host.y + dir_y[i] >= 0 && host.y + dir_y[i] < size_y &&
				map[host.y + dir_y[i]][host.x + dir_x[i]] != -1 &&
				(!same(host, cells[map[host.y + dir_y[i]][host.x + dir_x[i]]]) || check(host.genome.cannibalism / 2))) {
				avdir.push_back(i);
			}
		}

		if (avdir.size() != 0) {
			short cash;

			short dir_i = choose_dir(gen);
			dir_i = dir_i % avdir.size();

			cell& target = cells[map[host.y + dir_y[avdir[dir_i]]][host.x + dir_x[avdir[dir_i]]]];

			cash = map[host.y][host.x];
			map[host.y][host.x] = -1;

			target.alive = false;

			host.y += dir_y[avdir[dir_i]]; host.x += dir_x[avdir[dir_i]];
			map[host.y][host.x] = cash;

			host.energy -= target.genome.energy_limit / sqrt(host.genome.consuming);		

			if (check(host.genome.energy_limit + host.genome.consuming / target.genome.energy_limit)) // Р Р†РЎР‚Р ВµР СР ВµР Р…Р Р…Р С•Р Вµ
				host.energy += target.energy;
			// 15 + 1


			return true;
		}
		else return false;
	}

	bool product(short host_id)
	{
		cell& host = cells[host_id];

		if (host.genome.production > 120) {
			host.energy += sqrt(host.genome.production);
			return true;
		}
		else if (host.genome.production > 90) {
			host.energy += sqrt(host.genome.production);
			return false;
		}
		else
			return false;
	}

	void born(short parent_id, short xb, short yb)
	{
		cell& parent = cells[parent_id];

		cells.push_back(Adam);
		cell& child = cells[cells.size() - 1];

		child.genome.age_breed = parent.genome.age_breed + radiance();
		child.genome.age_limit = parent.genome.age_limit + radiance();
		child.genome.connecting = parent.genome.connecting + radiance();
		child.genome.production = parent.genome.production + radiance();
		child.bh_child.product += (child.genome.production - parent.genome.production);
		child.bh_adult.product += (child.genome.production - parent.genome.production);
		child.genome.consuming = 255 - child.genome.production;
		child.bh_child.consume += (child.genome.consuming - parent.genome.consuming);
		child.bh_adult.consume += (child.genome.consuming - parent.genome.consuming);
		child.genome.cannibalism = parent.genome.cannibalism + radiance();
		child.genome.energy_aware = parent.genome.energy_aware + radiance();
		child.genome.energy_limit = parent.genome.energy_limit + radiance() * 2;
		child.genome.fertility = parent.genome.fertility + radiance();

		child.bh_adult = parent.bh_adult; child.bh_child = child.bh_child;

		// ACHTUNG ZONEN \/ \/ \/

		child.bh_child.consume = parent.bh_child.consume + radiance();
		child.bh_child.product = parent.bh_child.product + radiance();
		child.bh_child.move = parent.bh_child.move + radiance();
		child.bh_child.save = parent.bh_child.save + radiance();
		child.bh_child.share = parent.bh_child.share + radiance();

		child.bh_adult.consume = parent.bh_adult.consume + radiance();
		child.bh_adult.product = parent.bh_adult.product + radiance();
		child.bh_adult.move = parent.bh_adult.move + radiance();
		child.bh_adult.save = parent.bh_adult.save + radiance();
		child.bh_adult.share = parent.bh_adult.share + radiance();
		child.bh_adult.breed = parent.bh_adult.breed + radiance();

		// ACHTUNG ZONEN /\ /\ /\

		if (child.genome.age_breed > 90) child.genome.age_breed = 90;
		else if (child.genome.age_breed < 1) child.genome.age_breed = 1;
		if (child.genome.age_limit > 100) child.genome.age_limit = 100;
		else if (child.genome.age_limit < 5) child.genome.age_limit = 5;
		if (child.genome.connecting > 255) child.genome.connecting = 255;
		else if (child.genome.connecting < 1) child.genome.connecting = 1;

		if (child.genome.production > 255) child.genome.production = 255;
		else if (child.genome.production < 1) child.genome.production = 1;
		if (child.genome.consuming > 255) child.genome.consuming = 255;
		else if (child.genome.consuming < 1) child.genome.consuming = 1;
		if (child.genome.cannibalism > 255) child.genome.cannibalism = 255;
		else if (child.genome.cannibalism < 1) child.genome.cannibalism = 1;

		if (child.genome.energy_aware > 750) child.genome.energy_aware = 750;
		else if (child.genome.energy_aware < 50) child.genome.energy_aware = 50;
		if (child.genome.energy_limit > 1000) child.genome.energy_limit = 1000;
		else if (child.genome.energy_limit < 100) child.genome.energy_limit = 100;
		if (child.genome.fertility > 255) child.genome.fertility = 255;
		else if (child.genome.fertility < 1) child.genome.fertility = 1;

		parent.energy -= parent.genome.energy_limit / parent.genome.fertility;
		child.energy = parent.genome.energy_limit / parent.genome.fertility;

		child.x = xb; child.y = yb;
		map[yb][xb] = cells.size() - 1;
	}

	bool breed(short host_id)
	{
		cell& host = cells[host_id];

		vector<short> avdir;

		for (short i = 0; i <= 3; i++) {
			if (host.x + dir_x[i] >= 0 && host.x + dir_x[i] < size_x &&
				host.y + dir_y[i] >= 0 && host.y + dir_y[i] < size_y &&
				map[host.y + dir_y[i]][host.x + dir_x[i]] == -1) {
				avdir.push_back(i);
			}
		}

		if (avdir.size() != 0) {

			short dir_i = choose_dir(gen);
			dir_i = dir_i % avdir.size();

			if (map[host.y + dir_y[avdir[dir_i]]][host.x + dir_x[avdir[dir_i]]] == -1) {

				born(host_id, host.x + dir_x[avdir[dir_i]], host.y + dir_y[avdir[dir_i]]);
				return true;
			}
		}
		else return false;
	}

	bool share(short host_id)
	{
		cell& host = cells[host_id];

		vector<short>avdir;


		for (short i = 0; i <= 3; i++) {
			if (host.x + dir_x[i] >= 0 && host.x + dir_x[i] < size_x &&
				host.y + dir_y[i] >= 0 && host.y + dir_y[i] < size_y &&
				map[host.y + dir_y[i]][host.x + dir_x[i]] != -1 &&
				!same(host, cells[map[host.y + dir_y[i]][host.x + dir_x[i]]])) {
				avdir.push_back(i);
			}
		}

		if (avdir.size() != 0) {

			short dir_i = choose_dir(gen);
			dir_i = dir_i % avdir.size();

			cell& target = cells[map[host.y + dir_y[avdir[dir_i]]][host.x + dir_x[avdir[dir_i]]]];

			host.energy -= host.genome.connecting;
			target.energy += host.genome.connecting;

			return true;
		}
		else return false;
	}

	void action(short host_id)
	{
		cell& host = cells[host_id];

		if (host.bh_child.consume < 1) host.bh_child.consume = 1.0f; else if (host.bh_child.consume > 100) host.bh_child.consume = 100;
		if (host.bh_child.move < 1) host.bh_child.move = 1.0f; else if (host.bh_child.move > 100) host.bh_child.move = 100;
		if (host.bh_child.product < 1) host.bh_child.product = 1.0f; else if (host.bh_child.product > 100) host.bh_child.product = 100;
		if (host.bh_child.save < 1) host.bh_child.save = 1.0f; else if (host.bh_child.save > 100) host.bh_child.save = 100;
		if (host.bh_child.share < 1) host.bh_child.share = 1.0f; else if (host.bh_child.share > 100) host.bh_child.share = 100;

		if (host.bh_adult.consume < 1) host.bh_adult.consume = 1.0f; else if (host.bh_adult.consume > 100) host.bh_adult.consume = 100;
		if (host.bh_adult.move < 1) host.bh_adult.move = 1.0f; else if (host.bh_adult.move > 100) host.bh_adult.move = 100;
		if (host.bh_adult.product < 1) host.bh_adult.product = 1.0f; else if (host.bh_adult.product > 100) host.bh_adult.product = 100;
		if (host.bh_adult.save < 1) host.bh_adult.save = 1.0f; else if (host.bh_adult.save > 100) host.bh_adult.save = 100;
		if (host.bh_adult.share < 1) host.bh_adult.share = 1.0f; else if (host.bh_adult.share > 100) host.bh_adult.share = 100;
		if (host.bh_adult.breed < 1) host.bh_adult.breed = 1.0f; else if (host.bh_adult.breed > 100) host.bh_adult.breed = 100;

		host.age++;

		if (host.age > host.genome.age_limit || host.energy < 0) {
			host.alive = false;
			map[host.y][host.x] = -1;
			return;
		}

		else if (host.age > 0 && host.energy > 0 && host.age < host.genome.age_limit)
		{
			if (host.energy < host.genome.energy_aware) {
				if (product(host_id))
					return;

				else if (consume(host_id))
					return;

				else if (move(host_id))
					return;
			}

			if (host.energy > host.genome.energy_limit / 2) {

				if (breed(host_id))
					return;
			}

			// CHILD
			if (host.age < host.genome.age_breed)
			{
				while (true) {
					if (check(host.bh_child.product) && product(host_id)) {
						/*
						host.bh_child.product += 1.0f;

						host.bh_child.consume -= 0.25f;
						host.bh_child.save -= 0.25f;
						host.bh_child.move -= 0.25f;
						host.bh_child.share -= 0.25f;
						*/
						return;
					}

					else if (check(host.bh_child.consume) && consume(host_id)) {

						/*host.bh_child.consume += 1.0f;

						host.bh_child.product -= 0.25f;
						host.bh_child.save -= 0.25f;
						host.bh_child.move -= 0.25f;
						host.bh_child.share -= 0.25f;*/

						return;
					}

					else if (check(host.bh_child.save)) {

						host.energy -= 4;

						/*host.bh_child.save += 1.0f;

						host.bh_child.consume -= 0.25f;
						host.bh_child.product -= 0.25f;
						host.bh_child.move -= 0.25f;
						host.bh_child.share -= 0.25f;*/

						return;
					}

					else if (check(host.bh_child.move) && move(host_id)) {

						/*host.bh_child.move += 1.0f;

						host.bh_child.product -= 0.25f;
						host.bh_child.save -= 0.25f;
						host.bh_child.consume -= 0.25f;
						host.bh_child.share -= 0.25f;*/

						return;
					}

					else if (host.genome.connecting < host.energy && check(host.bh_child.share) && share(host_id)) {

						/*host.bh_child.share += 1.0f;

						host.bh_child.product -= 0.25f;
						host.bh_child.save -= 0.25f;
						host.bh_child.consume -= 0.25f;
						host.bh_child.move -= 0.25f;*/

						return;
					}
				}
			}

			// ADULT
			else
			{
				while (true) {
					if (check(host.bh_adult.product) && product(host_id)) {

						/*host.bh_adult.product += 1.0f;

						host.bh_adult.consume -= 0.2f;
						host.bh_adult.save -= 0.2f;
						host.bh_adult.move -= 0.2f;
						host.bh_adult.share -= 0.2f;
						host.bh_adult.breed -= 0.2f;*/

						return;
					}

					else if (check(host.bh_adult.consume) && consume(host_id)) {

						/*host.bh_adult.consume += 1.0f;

						host.bh_adult.product -= 0.2f;
						host.bh_adult.save -= 0.2f;
						host.bh_adult.move -= 0.2f;
						host.bh_adult.share -= 0.2f;
						host.bh_adult.breed -= 0.2f;*/

						return;
					}

					else if (check(host.bh_adult.save)) {

						host.energy -= 5;

						/*host.bh_adult.save += 1.0f;

						host.bh_adult.product -= 0.2f;
						host.bh_adult.consume -= 0.2f;
						host.bh_adult.move -= 0.2f;
						host.bh_adult.share -= 0.2f;
						host.bh_adult.breed -= 0.2f;*/

						return;
					}

					else if (check(host.bh_adult.move) && move(host_id)) {

						/*host.bh_adult.move += 1.0f;

						host.bh_adult.product -= 0.2f;
						host.bh_adult.save -= 0.2f;
						host.bh_adult.consume -= 0.2f;
						host.bh_adult.share -= 0.2f;
						host.bh_adult.breed -= 0.2f;*/

						return;
					}

					else if (host.genome.connecting < host.energy && check(host.bh_adult.share) && share(host_id)) {

						/*host.bh_adult.share += 1.0f;

						host.bh_adult.product -= 0.2f;
						host.bh_adult.save -= 0.2f;
						host.bh_adult.move -= 0.2f;
						host.bh_adult.consume -= 0.2f;
						host.bh_adult.breed -= 0.2f;*/

						return;
					}

					else if (check(host.bh_adult.breed && host.energy > host.genome.energy_limit / 2) && breed(host_id)) {

						/*host.bh_adult.breed += 1.0f;

						host.bh_adult.product -= 0.2f;
						host.bh_adult.save -= 0.2f;
						host.bh_adult.move -= 0.2f;
						host.bh_adult.share -= 0.2f;
						host.bh_adult.consume -= 0.2f;*/

						return;
					}
				}
			}


		}
	}

	void soup()
	{
		for (short i = 0; i < cells.size(); i++) {
			if (cells[i].alive)
				action(i);
		}

		for (short i = 0; i < cells.size(); i++) {
			if (i >= 0 && !cells[i].alive) {
				cells.erase(cells.begin() + i);
				i--;
			}
		}

		for (short i = 0; i < cells.size(); i++) {
			map[cells[i].y][cells[i].x] = i;
		}
	}

	void view()
	{
		system("cls");
		stringstream out;

		for (short y = 0; y < size_y; y++) {
			for (short x = 0; x < size_x; x++) {
				if (map[y][x] == -1)

					out << ".";
				else
					out << "\x1b[38;2;" << cells[map[y][x]].genome.energy_limit / 3 << ";" << cells[map[y][x]].genome.production << ";" << cells[map[y][x]].genome.fertility << "m@\x1b[0m";;
			}
			out << "\n";
		}
		cout << out.str();
	}
};

void main()
{
	Ecosystem Test;

	while (true)
	{
		Test.view();


		Test.soup();


		Sleep(20);
	}
}
```
