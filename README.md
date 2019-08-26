How to count tabular and map datasets in a CKAN instance using the API, as a
Jupyter Notebook.

## Quick start

Make sure you have either [Jupyter](https://jupyter.org/) Notebook or Lab
installed. Install the [Jupytext](https://github.com/mwouts/jupytext)
extension -- this is useful for Jupyter Notebooks to play nice with Git's
line based version control, and required to open `.md` (or `.py`files) as
Notebooks inside Jupyter. Learn more about Jupytext on this
[Towards Data Science blog post](https://towardsdatascience.com/introducing-jupytext-9234fdff6c57).

Create a new Python environtment and activate it

```
$ python -m venv env
$ source env/bin/activate
```

Inside the environment, install `ipykernel` and the new environment inside
Jupyter.

```
pip install ipykernel
python -m ipykernel install --user --name=ckanapi --display-name="CKAN API"
```

Now install the requirements for this specific notebook inside the new
environment. In particular, we use just `ckanapi`, a
[Python wrapper](https://github.com/ckan/ckanapi) for the
[CKAN API](https://docs.ckan.org/en/2.8/api/index.html), and
[tqdm](https://github.com/tqdm/tqdm) to display a nice progress bar.

```
pip install -r requirements.txt
```

Now you can just open the Jupyter Notebook provided and run it. Adjust the
parameters for your CKAN instance and enjoy!

