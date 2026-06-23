108 行类别是什么

真实机器人上没有 semantic_sensor 输出的完美实例分割图。论文/代码之所以这么设计，是因为这个工作的核心贡献是导航+探索策略，不是目标识别。用 GT 语义做 grounding 可以排除感知错误的干扰，单独评测导航能力。如果目标 grounding 都错了，那导航失败也分不清是感知不行还是导航不行。


有一个类别的text
        target_obj_id_mapping = {}
        if semantic_obs is not None:
            for target_gt_id in gt_target_obj_ids:
                target_obj_mask = semantic_obs == target_gt_id
                if (
                    np.sum(target_obj_mask)
                    / (target_obj_mask.shape[0] * target_obj_mask.shape[1])
                    > 0.0001
                ):
                    # loop through the detected objects to find the highest IoU with the target object
                    max_iou = -1
                    max_iou_obj_id = None
                    for idx, obj_id in enumerate(detection_list.keys()):
                        detected_mask = gobs["mask"][idx]
                        iou_score = IoU(detected_mask, target_obj_mask)
                        if iou_score > max_iou:
                            max_iou = iou_score
                            max_iou_obj_id = obj_id
                    if max_iou > self.cfg.scene_graph.target_obj_iou_threshold:
                        target_obj_id_mapping[target_gt_id] = max_iou_obj_id
                        logging.info(
                            f"Target object {target_gt_id} detected with IoU {max_iou} in {img_path}!!!"
                        )

                        问题很大