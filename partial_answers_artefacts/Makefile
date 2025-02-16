target_outputs=newspaper ncompas ncountry month year subject geographic all
bert_models=$(patsubst %,data/models/bert_cls_1_uniform_2_shallow_512_body_%_clean.pt,$(target_outputs))

clean:
	rm -rf data/final

assert_top:
	@echo "Make sure you're running me from the top-level directory, so that data will be stored in 'data/final/' [Ctrl-C to exit, ENTER to continue]"
	@read _

_data:
	mkdir -p computed data/final/ data/models/ data/embeddings/ data/eval/ data/misc/
	wget --user finalproject --password Rik@LfD21 -P data/final/ https://teaching.stijneikelboom.nl/lfd2122/COP.filt3.sub.zip
	unzip -j -d data/final/ data/final/COP.filt3.sub.zip

	# join all JSON files together in a hacky way
	echo "[" > data/final/all.json
	for f in data/final/COP*.filt3.sub.json ; do \
		cat $$f ; \
		echo -n "," ; \
	done >> data/final/all.json;
	echo "]" >> data/final/all.json

	# remove trailing comma
	sed 's/,]/]/' -i data/final/all.json

	# remove temporary files
	rm data/final/COP*.filt3.sub.json
	rm data/final/COP.filt3.sub.zip

data: assert_top _data

data_all: assert_top _data prepare_data craft_data

prepare_data:
	python3 ./src/data/prepare_clean.py --data-in data/final/all.json --data-out data/final/clean.json

craft_data:
	python3 ./src/data/craft.py --data-in data/final/clean.json --data-out data/final/{LABEL}.json

get_glove:
	wget -P data/ http://nlp.stanford.edu/data/wordvecs/glove.6B.zip
	unzip -d data/ data/glove.6B.zip
	python3 ./src/data/prepare_glove.py --data-in data/glove.6B.200d.txt --data-out data/glove.pkl

run_rv1:
	python3 src/train_lm_classifier.py -to craft -ti craft -ep 2 --max-length 512 -bs 8 --input data/final/Rv1_$(dr)_n.json
	python3 src/train_lm_classifier.py -to craft -ti craft -ep 2 --max-length 512 -bs 8 --input data/final/Rv1_$(dr)_c.json
	python3 src/train_lm_classifier.py -to craft -ti craft -ep 2 --max-length 512 -bs 8 --input data/final/Rv1_$(dr)_o.json
	python3 src/train_lm_classifier.py -to craft -ti craft -ep 2 --max-length 512 -bs 8 --input data/final/Rv1_$(dr)_m.json
	python3 src/train_lm_classifier.py -to craft -ti craft -ep 2 --max-length 512 -bs 8 --input data/final/Rv1_$(dr)_y.json
	python3 src/train_lm_classifier.py -to craft -ti craft -ep 2 --max-length 512 -bs 8 --input data/final/Rv1_$(dr)_s.json
	python3 src/train_lm_classifier.py -to craft -ti craft -ep 2 --max-length 512 -bs 8 --input data/final/Rv1_$(dr)_g.json

tsne:
	python3 src/model_main/embedd.py -i data/final/clean.json -o data/embeddings/embeddings_{m}_{t}_{ml}.pkl -t body --max-length 512 -st all
	python3 src/data/project_dim.py -i data/embeddings/embeddings_bert_body_512.pkl --em avg
	python3 src/figures/tsne_plot.py -i data/final/clean.json -e data/embeddings/embeddings_bert_body_512_reduced_avg_tsne_2.pkl -l ncountry

balance:
	python3 src/data/balance.py -i data/final/clean.json

feature_dependency:
	python3 src/misc/feature_dependency.py -i data/final/clean.json --logfile computed/feature_dependency.out
	python3 src/figures/feature_dependency_lr.py --logfile computed/feature_dependency.out

meta_model_source:
	echo "TODO"

meta_model:
	echo "TODO"

get_all_0v1:
	wget -P data/models/ "vilda.net/t/bert_cls_1_uniform_2_shallow_512_body_all_clean.pt"

train_%_0v1:
	python3 src/model_main/train_lm_classifier.py -to $(patsubst train_%_0v1,%,$@)

train_0v1: $(patsubst %,train_%_0v1,$(target_outputs))

data/models/bert_cls_1_uniform_2_shallow_512_body_%_clean.pt: train_%_0v1

eval_%_0v1: 
	python3 src/model_main/eval_lm_classifier.py --model-path $(patsubst eval_%_0v1,data/models/bert_cls_1_uniform_2_shallow_512_body_%_clean.pt,$@)  -to $(patsubst eval_%_0v1,%,$@)

eval_0v1: $(bert_models) $(patsubst %,eval_%_0v1,$(target_outputs))

traineval_0v1: train_0v1 eval_0v1

train_1v1:
	@echo "This script will generate a large workload. [Ctrl-C to exit, ENTER to continue]"
	@read _
	files=(data/final/1v1_*) \
	for (( index=0; index<${#files[@]}; index+=1 )); do \
		f=${files[index]}; \
		echo $$f; \
		python3 ./src/train_lm_classifier.py -to craft -ti craft -ep 2 --max-length 512 -bs 8 --input $$f; \
	done \
    
prettify_eval:
	for f in data/eval/*.json; do \
		echo $$f; \
		cat $$f | python3 -m json.tool > tmp.json; \
		mv tmp.json $$f; \
	done