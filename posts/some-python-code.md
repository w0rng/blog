+++
date = '2025-03-09T15:01:13+07:00'
draft = true
title = 'Some Python Code'
+++

Если нужно изменить данные в pk поле в django, может стать больно. Особенно, если есть связи. Вот миграция, которая это делает (скорее всего надо будет допилить под себя):
```
from django.db import connection, migrations, models


def update_m2m(relations, Sign):
    for entry in relations.objects.all():
        if entry.newsign_id is None:
            continue
        old_id = entry.newsign_id
        new_sign = Sign.objects.get(old_id=old_id)
        entry.newsign_id = new_sign.id
        entry.save()


def update_o2m(relations, Sign):
    print('Restoring relations for:', relations)
    for meaning in relations.objects.all():
        if meaning.sign_id is None:
            continue
        old_id = meaning.sign_id
        new_sign = Sign.objects.get(old_id=old_id)
        meaning.sign_id = new_sign.id
        meaning.save()
    print('Restored')


def restore_relations(apps, schema_editor):
    Sign = apps.get_model('meaning', 'NewSign')

    update_o2m(apps.get_model('meaning', 'Meaning'), Sign)
    update_o2m(apps.get_model('animation', 'WebGLAnimation'), Sign)

    print('Restoring relations for datasettextlinetranslation_signs')
    update_m2m(apps.get_model('dataset', 'datasettextlinetranslation_signs'), Sign)
    print('Restored')
    print('Restoring relations for newsign_comments')
    update_m2m(apps.get_model('meaning', 'newsign_comments'), Sign)
    print('Restored')
    print('Restoring relations for newsign_webgl_animations')
    update_m2m(apps.get_model('meaning', 'newsign_webgl_animations'), Sign)
    print('Restored')
    print('Restoring relations for newsign_tags')
    update_m2m(apps.get_model('meaning', 'newsign_tags'), Sign)
    print('Restored')


constrains = {}


def get_constraints(table_name):
    with connection.cursor() as cursor:
        cursor.execute(
            f"""
            SELECT constraint_name
            FROM information_schema.table_constraints
            WHERE table_name='{table_name}' AND constraint_type='UNIQUE';
            """,
        )
        constrains[table_name] = [row[0] for row in cursor.fetchall()]
        return constrains[table_name]


def drop_constraints(apps, schema_editor):
    tables = {
        'dataset_datasettextlinetranslation_signs',
        'meaning_newsign_comments',
        'meaning_newsign_tags',
        'meaning_newsign_webgl_animations',
    }

    with connection.cursor() as cursor:
        for table in tables:
            constraints = get_constraints(table)
            for constraint in constraints:
                print('Dropping constraint:', constraint)
                cursor.execute(f'ALTER TABLE {table} DROP CONSTRAINT {constraint};')
                print('Constraint dropped')


def restore_constraints(apps, schema_editor):
    constraints_sql = {
        'dataset_datasettextlinetranslation_signs': 'ALTER TABLE dataset_datasettextlinetranslation_signs ADD CONSTRAINT %s UNIQUE (newsign_id, datasettextlinetranslation_id);',
        'meaning_newsign_comments': 'ALTER TABLE meaning_newsign_comments ADD CONSTRAINT %s UNIQUE (newsign_id, comment_id);',
        'meaning_newsign_webgl_animations': 'ALTER TABLE meaning_newsign_webgl_animations ADD CONSTRAINT %s UNIQUE (newsign_id, animationasset_id);',
        'meaning_newsign_tags': 'ALTER TABLE meaning_newsign_tags ADD CONSTRAINT %s UNIQUE (newsign_id, tag_id);',
    }

    with connection.cursor() as cursor:
        for table, constrain in constraints_sql.items():
            for name in constrains[table]:
                print('Restoring constraint for:', table)
                cursor.execute(constrain % name)
                print('Constraint restored')


class Migration(migrations.Migration):
    dependencies = [
        ('meaning', '0019_rename_webgl_animations_new_newsign_webgl_animations'),
    ]

    operations = [
        migrations.AddField(
            model_name='newsign',
            name='old_id',
            field=models.BigIntegerField(null=True, blank=True),
        ),
        migrations.RunSQL(
            sql='UPDATE meaning_newsign SET old_id = id;',
            reverse_sql=migrations.RunSQL.noop,
        ),
        migrations.RemoveField(
            model_name='newsign',
            name='id',
        ),
        migrations.RenameField(
            model_name='newsign',
            old_name='number',
            new_name='id',
        ),
        migrations.AlterField(
            model_name='newsign',
            name='id',
            field=models.AutoField(primary_key=True),
        ),
        migrations.RunPython(drop_constraints),
        migrations.RunPython(restore_relations),
        migrations.RunPython(restore_constraints),
        migrations.RemoveField(
            model_name='newsign',
            name='old_id',
        ),
    ]
```
