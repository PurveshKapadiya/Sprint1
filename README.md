# Repository Name Sprint1
# Organization Name
# Deep Panwala
# Purvesh Kapadiya

import re
from datetime import datetime
from prettytable import PrettyTable

individual_pattern = re.compile(r'0 @I(\d+)@ INDI')
name_pattern = re.compile(r'1 NAME (.+)')
gender_pattern = re.compile(r'1 SEX ([MF])')
birthday_pattern = re.compile(r'1 BIRT\n2 DATE (\d{1,2} [A-Z]{3} \d{4})')
death_pattern = re.compile(r'1 DEAT\n2 DATE (\d{1,2} [A-Z]{3} \d{4})')
family_child_pattern = re.compile(r'1 FAMC @F(\d+)@')
family_spouse_pattern = re.compile(r'1 FAMS @F(\d+)@')

family_pattern = re.compile(r'0 @F(\d+)@ FAM')
marriage_pattern = re.compile(r'1 MARR\n2 DATE (\d{1,2} [A-Z]{3} \d{4})')
divorce_pattern = re.compile(r'1 DIV\n2 DATE (\d{1,2} [A-Z]{3} \d{4})')
husband_pattern = re.compile(r'1 HUSB @I(\d+)@')
wife_pattern = re.compile(r'1 WIFE @I(\d+)@')
child_pattern = re.compile(r'1 CHIL @I(\d+)@')

individuals = {}
families = {}
current_individual = None
current_family = None

def calculate_age(birthdate):
    birthdate = datetime.strptime(birthdate, '%d %b %Y')
    end_date = datetime(2030, 9, 12)
    age = end_date.year - birthdate.year - ((end_date.month, end_date.day) < (birthdate.month, birthdate.day))
    return age



with open("C:\export-Forest.ged", 'r') as gedcom_file:
    lines = gedcom_file.readlines()

for line in lines:
    individual_match = individual_pattern.match(line)
    if individual_match:
        if current_individual:
            individuals[current_individual['identifier']] = current_individual
        current_individual = {
            'identifier': individual_match.group(1),
            'name': None,
            'gender': None,
            'birthday': None,
            'age': None,
            'alive': True,
            'death': None,
            'child': [],
            'spouse': [],
        }

    name_match = name_pattern.match(line)
    if name_match and current_individual:
        current_individual['name'] = name_match.group(1)

    gender_match = gender_pattern.match(line)
    if gender_match and current_individual:
        current_individual['gender'] = 'Male' if gender_match.group(1) == 'M' else 'Female'

    birthday_match = birthday_pattern.match(line)
    if birthday_match and current_individual:
        current_individual['birthday'] = birthday_match.group(1)
        current_individual['age'] = calculate_age(birthday_match.group(1))

    death_match = death_pattern.match(line)
    if death_match and current_individual:
        current_individual['alive'] = False
        current_individual['death'] = death_match.group(1)

    family_child_match = family_child_pattern.match(line)
    if family_child_match and current_individual:
        current_individual['child'].append(family_child_match.group(1))

    family_spouse_match = family_spouse_pattern.match(line)
    if family_spouse_match and current_individual:
        current_individual['spouse'].append(family_spouse_match.group(1))
    
       family_match = family_pattern.match(line)
    if family_match:
        if current_family:
            families[current_family['identifier']] = current_family
        current_family = {
            'identifier': family_match.group(1),
            'marriage_date': None,
            'divorce_date': None,
            'husband_id': None,
            'husband_name': None,
            'wife_id': None,
            'wife_name': None,
            'children': [],
        }

    marriage_match = marriage_pattern.match(line)
    if marriage_match and current_family:
        current_family['marriage_date'] = marriage_match.group(1)

    divorce_match = divorce_pattern.match(line)
    if divorce_match and current_family:
        current_family['divorce_date'] = divorce_match.group(1)

    husband_match = husband_pattern.match(line)
    if husband_match and current_family:
        current_family['husband_id'] = husband_match.group(1)
        current_family['husband_name'] = individuals.get(husband_match.group(1), {}).get('name', 'Unknown')

    wife_match = wife_pattern.match(line)
    if wife_match and current_family:
        current_family['wife_id'] = wife_match.group(1)
        current_family['wife_name'] = individuals.get(wife_match.group(1), {}).get('name', 'Unknown')

    child_match = child_pattern.match(line)
    if child_match and current_family:
        current_family['children'].append(child_match.group(1))

individual_table = PrettyTable()
individual_table.field_names =["Identifier", "Name", "Gender", "Birthday", "Age", "Alive", "Death", "Child", "Spouse"]

for identifier, data in sorted(individuals.items(), key=lambda x: int(x[0])):
    individual_table.add_row([
        data['identifier'],
        data['name'],
        data['gender'],
        data['birthday'],
        data['age'],
        "Yes" if data['alive'] else "No",
        data['death'] if not data['alive'] else "",
        ", ".join(data['child']),
        ", ".join(data['spouse'])
    ])

family_table = PrettyTable()
family_table.field_names = ["Family Identifier", "Married", "Divorced", "Husband ID", "Husband Name", "Wife ID",
                            "Wife Name", "Children"]

for identifier, data in sorted(families.items(), key=lambda x: int(x[0])):
    family_table.add_row([
        data['identifier'],
        data['marriage_date'],
        data['divorce_date'],
        data['husband_id'],
        data['husband_name'],
        data['wife_id'],
        data['wife_name'],
        ", ".join(data['children'])
    ])

print("Individuals Table:")
print(individual_table)

print("\nFamilies Table:")
print(family_table)

# Define user story functions
def us01_check_dates(individuals):
    current_date = datetime(2030, 9, 12)
    errors = []
    for indi_id, indi_data in individuals.items():
        if indi_data['birthday']:
            birth_date = datetime.strptime(indi_data['birthday'], '%d %b %Y')
            if birth_date > current_date:
                errors.append(f"ERROR: INDIVIDUAL: US01: {indi_id}: Birthday {birth_date.strftime('%Y-%m-%d')} occurs in the future")
    return errors

def us02_check_marriage_dates(families, individuals):
    errors = []
    for fam_id, fam_data in families.items():
        marriage_date = datetime.strptime(fam_data['marriage_date'], '%d %b %Y') if fam_data['marriage_date'] else None
        if marriage_date:
            husband_birth_date = datetime.strptime(individuals[fam_data['husband_id']]['birthday'], '%d %b %Y')
            wife_birth_date = datetime.strptime(individuals[fam_data['wife_id']]['birthday'], '%d %b %Y')
            if marriage_date < husband_birth_date:
                errors.append(f"ERROR: FAMILY: US02: {fam_id}: Husband's birth date {husband_birth_date.strftime('%Y-%m-%d')} is after marriage date {marriage_date.strftime('%Y-%m-%d')}")
            if marriage_date < wife_birth_date:
                errors.append(f"ERROR: FAMILY: US02: {fam_id}: Wife's birth date {wife_birth_date.strftime('%Y-%m-%d')} is after marriage date {marriage_date.strftime('%Y-%m-%d')}")
    return errors
    
